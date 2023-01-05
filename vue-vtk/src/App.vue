
<template>
  <div>
    <div ref="vtkContainer" />
    <table class="controls">
      <tbody>
        <tr>
          <td>
            {{ scale }}
          </td>
        </tr>
        <tr>
          <td>
            <input
              type="range"
              min="0"
              max="4"
              step="0.25"
              :value="scale"
              @input="setScale($event.target.value)"
            />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { ref, unref, onMounted, onBeforeUnmount, watchEffect } from 'vue';

import '@kitware/vtk.js/Rendering/Profiles/Geometry';

import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';

import macro from '@kitware/vtk.js/macros';
import vtk from '@kitware/vtk.js/vtk';
import vtkActor           from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper          from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkPlaneSource     from '@kitware/vtk.js/Filters/Sources/PlaneSource';
import AxesActor from '@kitware/vtk.js/Rendering/Core/AxesActor';
import vtkSTLReader       from '@kitware/vtk.js/IO/Geometry/STLReader';
import vtkOutlineFilter   from '@kitware/vtk.js/Filters/General/OutlineFilter';
import vtkContourTriangulator from '@kitware/vtk.js/Filters/General/ContourTriangulator';
import vtkDataArray from '@kitware/vtk.js/Common/Core/DataArray';
import vtkWarpScalar from '@kitware/vtk.js/Filters/General/WarpScalar';
import stlUrl             from './assets/beam_mesh.stl?url'

export default {
  name: 'App',

  setup() {
    const vtkContainer = ref(null);
    const context = ref(null);
    const scale = ref(1);

    function setScale(sc) {
      scale.value = Number(sc);
    }

    watchEffect(() => {
      const sc = unref(scale);
      if (context.value) {
        const { renderWindow, warp } = context.value;
        warp.setScaleFactor(sc);
        warp.update();
        renderWindow.render();
      }
    });

    onMounted(async () => {
      if (!context.value) {
        const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
          rootContainer: vtkContainer.value,
        });
        const beam = vtkSTLReader.newInstance();
        await beam.setUrl(stlUrl);
        const coneSource = vtkPlaneSource.newInstance({point1: [5, 0, 0], point2: [0, 1, 0], xResolution: 50});
        const warp = vtkWarpScalar.newInstance({scaleFactor: 2, useNormal: true});

        const randFilter = macro.newInstance((publicAPI, model) => {
          macro.obj(publicAPI, model); // make it an object
          macro.algo(publicAPI, model, 1, 1); // mixin algorithm code 1 in, 1 out
          publicAPI.requestData = (inData, outData) => {
            // implement requestData
            if (!outData[0] || inData[0].getMTime() > outData[0].getMTime()) {
              const newArray = new Float32Array(
                inData[0].getPoints().getNumberOfPoints()
              );
              for (let i = 0; i < newArray.length; i++) {
                // newArray[i] = Math.cos(inData[0].getPoints().getPoint(i)[0]*2*Math.PI);
               newArray[i] = Math.cos(inData[0].getPoints().getPoint(i)[0]);
              }
              console.log(newArray);
              const da = vtkDataArray.newInstance({ name: 'spike', values: newArray });
              const newDataSet = vtk({ vtkClass: inData[0].getClassName() });
              newDataSet.shallowCopy(inData[0]);
              newDataSet.getPointData().setScalars(da);
              outData[0] = newDataSet;
            }
          };
        })();
        
        randFilter.setInputConnection(coneSource.getOutputPort());
        warp.setInputConnection(randFilter.getOutputPort());
        
        
        const mapper = vtkMapper.newInstance();
        mapper.setInputConnection(warp.getOutputPort());
        
        warp.setInputArrayToProcess(0, 'spike', 'PointData', 'Scalars');

        const actor = vtkActor.newInstance();
        actor.setMapper(mapper);

        const renderer = fullScreenRenderer.getRenderer();
        const renderWindow = fullScreenRenderer.getRenderWindow();

        renderer.addActor(actor);
        renderer.addActor(AxesActor.newInstance());
        renderer.resetCamera();
        renderWindow.render();

        context.value = {
          fullScreenRenderer,
          renderWindow,
          renderer,
          coneSource,
          actor,
          mapper,
          warp,
        };
      }
    });

    onBeforeUnmount(() => {
      if (context.value) {
        const { fullScreenRenderer, coneSource, actor, mapper } = context.value;
        actor.delete();
        mapper.delete();
        coneSource.delete();
        fullScreenRenderer.delete();
        context.value = null;
      }
    });

    return {
      vtkContainer,
      setScale,
      scale
    };
  }
}
</script>

<style scoped>
.controls {
  position: absolute;
  top: 25px;
  left: 25px;
  background: white;
  padding: 12px;
}
</style>