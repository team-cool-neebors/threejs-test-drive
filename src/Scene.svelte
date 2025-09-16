<script>
  import { T, useThrelte } from '@threlte/core';
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from '@threlte/extras';
  import { mergeBufferGeometries } from 'three-stdlib';

  export let south = 51.473846, west = 3.588512, north = 51.529870, east = 3.651855;

  let mergedGeometry = null;
  let groundSize = 200;
  let buildingMetadata = [];
  let selectedBuilding = null;

  // Access scene and camera via useThrelte
  const { scene, camera } = useThrelte();

  let raycaster = new THREE.Raycaster();
  let mouse = new THREE.Vector2();

  // Web Mercator projection (meters)
  function lonLatToMeters(lon, lat) {
    const x = (lon * 20037508.34) / 180;
    let y = Math.log(Math.tan((90 + lat) * Math.PI / 360)) / (Math.PI / 180);
    y = (y * 20037508.34) / 180;
    return { x, y };
  }

  // Parse OSM tags for building height (meters)
  function parseHeight(tags) {
    if (!tags) return null;
    if (tags.height) {
      const v = parseFloat(String(tags.height).replace(',', '.'));
      if (!Number.isNaN(v)) return Math.abs(v);
    }
    if (tags['building:levels']) {
      const lv = parseFloat(String(tags['building:levels']).replace(',', '.'));
      if (!Number.isNaN(lv)) return Math.abs(lv) * 3.0;
    }
    if (tags['min_height']) {
      const v = parseFloat(String(tags['min_height']).replace(',', '.'));
      if (!Number.isNaN(v)) return Math.abs(v);
    }
    return null;
  }

  // Point-in-polygon test
  function isPointInPolygon(x, z, points) {
    let inside = false;
    for (let i = 0, j = points.length - 1; i < points.length; j = i++) {
      const xi = points[i][0], zi = points[i][1];
      const xj = points[j][0], zj = points[j][1];
      const intersect = ((zi > z) !== (zj > z)) && (x < (xj - xi) * (z - zi) / (zj - zi) + xi);
      if (intersect) inside = !inside;
    }
    return inside;
  }

  // Global center for the entire bounding box
  const globalCenterMerc = lonLatToMeters((west + east) / 2, (south + north) / 2);

  // Generate sub-boxes (2x2 grid)
  function generateSubBoxes() {
    const subBoxes = [];
    const latStep = (north - south) / 2;
    const lonStep = (east - west) / 2;
    for (let i = 0; i < 2; i++) {
      for (let j = 0; j < 2; j++) {
        const subSouth = south + i * latStep;
        const subNorth = subSouth + latStep;
        const subWest = west + j * lonStep;
        const subEast = subWest + lonStep;
        subBoxes.push({ south: subSouth, west: subWest, north: subNorth, east: subEast });
      }
    }
    return subBoxes;
  }

  // Fetch and process Overpass data for a single sub-box
  async function fetchSubBox({ south, west, north, east }, nodesCache, retries = 3, backoff = 1000) {
    const overpassQuery = `
      [out:json][timeout:25];
      (
        way["building"](${south},${west},${north},${east});
        relation["building"](${south},${west},${north},${east});
      );
      out body;
      >;
      out skel qt;
    `;
    const overpassUrl = `https://overpass-api.de/api/interpreter?data=${encodeURIComponent(overpassQuery)}`;

    try {
      const response = await fetch(overpassUrl);
      if (!response.ok) {
        if (response.status === 429) {
          console.log('Rate limited, waiting 60s');
          await new Promise(resolve => setTimeout(resolve, 60000));
          return fetchSubBox({ south, west, north, east }, nodesCache, retries, backoff);
        }
        if (response.status === 504 && retries > 0) {
          console.log(`504 Gateway Timeout, retrying in ${backoff}ms`);
          await new Promise(resolve => setTimeout(resolve, backoff));
          return fetchSubBox({ south, west, north, east }, nodesCache, retries - 1, backoff * 2);
        }
        throw new Error('Overpass fetch failed: ' + response.status);
      }
      const data = await response.json();

      for (const el of data.elements) {
        if (el.type === 'node') nodesCache[el.id] = el;
      }

      const subGeometries = [];
      const subMetadata = [];

      data.elements
        .filter((e) => e.type === 'way' && e.nodes && e.nodes.length >= 3)
        .forEach((way) => {
          const pts = [];
          let valid = true;
          const bounds = { minX: Infinity, maxX: -Infinity, minZ: Infinity, maxZ: -Infinity };
          for (const nid of way.nodes) {
            const n = nodesCache[nid];
            if (!n) {
              valid = false;
              break;
            }
            const m = lonLatToMeters(n.lon, n.lat);
            const x = m.x - globalCenterMerc.x;
            const z = m.y - globalCenterMerc.y;
            pts.push([x, z]);
            bounds.minX = Math.min(bounds.minX, x);
            bounds.maxX = Math.max(bounds.maxX, x);
            bounds.minZ = Math.min(bounds.minZ, z);
            bounds.maxZ = Math.max(bounds.maxZ, z);
          }
          if (!valid || pts.length < 3) return;

          // Close polygon if not closed
          const [fx, fy] = pts[0];
          const [lx, ly] = pts[pts.length - 1];
          if (fx !== lx || fy !== ly) pts.push([fx, fy]);

          // Create shape
          const shape = new THREE.Shape();
          shape.moveTo(pts[0][0], pts[0][1]);
          for (let i = 1; i < pts.length; i++) shape.lineTo(pts[i][0], pts[i][1]);

          // Extrude shape
          let h = parseHeight(way.tags) || (6 + Math.random() * 6);
          h = Math.min(Math.max(h, 1), 200);

          const extrudeSettings = { depth: h, bevelEnabled: false, steps: 1 };
          const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
          geometry.rotateX(-Math.PI / 2);

          subGeometries.push(geometry);
          subMetadata.push({ tags: way.tags || {}, bounds, height: h, points: pts });
        });

      return { geometries: subGeometries, metadata: subMetadata };
    } catch (error) {
      console.error('Error fetching sub-box:', error);
      return { geometries: [], metadata: [] };
    }
  }

  // Fetch all sub-boxes and merge
  async function loadOSMBuildings() {
    const subBoxes = generateSubBoxes();
    const allGeometries = [];
    const nodesCache = {};

    for (const box of subBoxes) {
      const { geometries, metadata } = await fetchSubBox(box, nodesCache);
      allGeometries.push(...geometries);
      buildingMetadata.push(...metadata);
      await new Promise(resolve => setTimeout(resolve, 5000)); // 5s delay
    }

    if (allGeometries.length > 0) {
      mergedGeometry = mergeBufferGeometries(allGeometries, false);
    }

    // Calculate scene bounds
    const box = new THREE.Box3();
    if (mergedGeometry) {
      const mesh = new THREE.Mesh(mergedGeometry);
      box.expandByObject(mesh);
    }
    const size = box.getSize(new THREE.Vector3());
    groundSize = Math.max(size.x, size.z, 200) * 2;
  }

  // Handle click for raycasting
  function onClick(event) {
    console.log('df');
    if (!scene || !camera.current) {
      console.log('Scene or camera not ready');
      return;
    }
    console.log('sd');

    // Convert mouse position to normalized device coordinates (-1 to +1)
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera.current);
    const intersects = raycaster.intersectObject(scene.getObjectByName('buildings'));

    if (intersects.length > 0) {
      const point = intersects[0].point;
      console.log('Intersection point:', point);
      // Find the building whose polygon contains the intersection point
      const clickedBuilding = buildingMetadata.find(meta => {
        // Add margin to bounding box to account for precision errors
        const margin = 1.0; // Adjust as needed
        if (
          point.x < meta.bounds.minX - margin ||
          point.x > meta.bounds.maxX + margin ||
          point.z < meta.bounds.minZ - margin ||
          point.z > meta.bounds.maxZ + margin ||
          point.y < -margin ||
          point.y > meta.height + margin
        ) {
          return false;
        }
        // Debug log bounds and points for comparison
        console.log('Checking building bounds:', meta.bounds, 'height:', meta.height, 'points:', meta.points);
        // Use point-in-polygon test
        return isPointInPolygon(point.x, point.z, meta.points);
      });

      if (clickedBuilding) {
        selectedBuilding = clickedBuilding.tags;
        console.log('Clicked building tags:', clickedBuilding.tags);
      } else {
        selectedBuilding = null;
        console.log('No building found at click point');
      }
    } else {
      selectedBuilding = null;
      console.log('No intersections');
    }
  }

  onMount(() => {
    loadOSMBuildings();
    window.addEventListener('click', onClick);
    return () => window.removeEventListener('click', onClick);
  });
</script>

<T.Scene>
  <T.PerspectiveCamera
    makeDefault
    fov={15}
    near={1}
    far={2e7}
    position={[0, Math.max(150, groundSize * 0.6), groundSize * 1.25]}
  >
    <OrbitControls enableDamping dampingFactor={0.06} />
  </T.PerspectiveCamera>

  <T.HemisphereLight color={0xffffff} groundColor={0x444444} intensity={0.6} />
  <T.DirectionalLight color={0xffffff} intensity={0.8} position={[500, 1000, 500]} castShadow />

  {#if mergedGeometry}
    <T.Mesh geometry={mergedGeometry} castShadow receiveShadow position={[0, 0, 0]} name="buildings">
      <T.MeshStandardMaterial color={0xdedede} roughness={0.9} metalness={0.05} />
    </T.Mesh>
  {/if}
</T.Scene>

{#if selectedBuilding}
  <div style="position: absolute; top: 10px; left: 10px; background: rgba(0, 0, 0, 0.7); color: white; padding: 10px; border-radius: 5px;">
    <h3>Building Tags</h3>
    {#each Object.entries(selectedBuilding) as [key, value]}
      <p>{key}: {value}</p>
    {/each}
  </div>
{/if}
