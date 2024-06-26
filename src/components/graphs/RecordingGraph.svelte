<!--
  (c) 2023, Center for Computational Thinking and Design at Aarhus University and contributors
 
  SPDX-License-Identifier: MIT
 -->

<script lang="ts">
  import { onMount } from 'svelte';
  import {
    Chart,
    registerables,
    ChartConfiguration,
    ChartTypeRegistry,
    LineController,
    LineElement,
    LinearScale,
    PointElement,
  } from 'chart.js';
  import RecordingInspector from '../3d-inspector/RecordingInspector.svelte';

  export let data: { x: number[]; y: number[]; z: number[] };

  let verticalLineX = NaN;
  let hoverIndex = NaN;
  let modalPosition = { x: 0, y: 0 };
  let modalSize = 250;
  const verticalLineCol = 'black';
  const verticalLineWidth = 1;

  const getDataByIndex = (index: number) => {
    if (isNaN(index)) {
      return { x: 0, y: 0, z: 0 };
    }
    return {
      x: data.x[index],
      y: data.y[index],
      z: data.z[index],
    };
  };

  let htmlElement: HTMLDivElement;
  const inspectorMarginPx = 5;

  function recomputeModalParams() {
    const rect = htmlElement.getBoundingClientRect();
    modalSize = generateSizeOfInspector(rect);
    modalPosition = generatePositionOfInspector(rect, modalSize);
  }

  /**
   * Positions in regards to size of modal following these rules:
   * - It prefers center-aligning with element in focus (The recording). Moves into view if circle
   * escapes the left or right side.
   * - It prefers showing downwards. Changes to upwards if downwards is not possible.
   * - It always positions at an offset from the element in focus.
   */
  function generatePositionOfInspector(
    rect: DOMRect,
    size: number,
  ): { x: number; y: number } {
    const rectCenterX = (rect.left + rect.right) / 2;

    let x = rectCenterX - size / 2;
    x = Math.max(0, x);
    x = Math.min(x, window.innerWidth - size);

    const showAboveGraph = window.innerHeight - rect.bottom - 15 < size;
    const y = showAboveGraph
      ? rect.top - inspectorMarginPx - size
      : rect.bottom + inspectorMarginPx;

    return { x, y };
  }

  function generateSizeOfInspector(rect: DOMRect): number {
    return (window.innerHeight - rect.height) / 2 - inspectorMarginPx;
  }

  function getConfig(): ChartConfiguration<
    keyof ChartTypeRegistry,
    { x: number; y: number }[],
    string
  > {
    const x: { x: number; y: number }[] = [];
    const y: { x: number; y: number }[] = [];
    const z: { x: number; y: number }[] = [];
    for (let i = 1; i < data.x.length; i++) {
      x.push({ x: i, y: data.x[i - 1] });
      y.push({ x: i, y: data.y[i - 1] });
      z.push({ x: i, y: data.z[i - 1] });
    }
    return {
      type: 'line',
      data: {
        datasets: [
          {
            label: 'x',
            borderColor: 'red',
            borderWidth: 1,
            pointRadius: 0,
            pointHoverRadius: 0,
            data: x,
          },
          {
            label: 'y',
            borderColor: 'green',
            borderWidth: 1,
            pointRadius: 0,
            pointHoverRadius: 0,
            data: y,
          },
          {
            label: 'z',
            borderColor: 'blue',
            borderWidth: 1,
            pointRadius: 0,
            pointHoverRadius: 0,
            data: z,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,

        interaction: {
          intersect: false,
          mode: 'index',
        },
        scales: {
          x: {
            type: 'linear',
            min: 0,
            max: data.x.length,
            grid: {
              color: '#f3f3f3',
            },
            ticks: {
              display: false, //this will remove only the label
            },
          },
          y: {
            type: 'linear',
            min: -2.5,
            max: 2.5,
            grid: {
              color: '#f3f3f3',
            },
            ticks: {
              display: false, //this will remove only the label
            },
          },
        },
        plugins: {
          legend: {
            display: false,
          },
          title: {
            display: false,
          },
        },
      },
      plugins: [
        {
          id: 'mouseLine',
          afterEvent: (chart, args) => {
            if (!args.inChartArea || args.event.type === 'mouseout') {
              verticalLineX = NaN;
              hoverIndex = NaN;
              return;
            }
            verticalLineX = args.event.x ?? NaN;
            if (args.event.native != null) {
              // only recompute modal params if modal is closed (i.e. this event opens the modal)
              if (isNaN(hoverIndex)) {
                recomputeModalParams();
              }
              hoverIndex = chart.getElementsAtEventForMode(
                args.event.native,
                'nearest',
                {},
                true,
              )[0].index;
            } else {
              hoverIndex = NaN;
            }
          },
          afterDraw: function (chart) {
            var ctx = chart.ctx;
            var chartArea = chart.chartArea;

            if (!isNaN(verticalLineX)) {
              ctx.save();
              ctx.strokeStyle = verticalLineCol;
              ctx.lineWidth = verticalLineWidth;
              let path = new Path2D();
              path.moveTo(verticalLineX, chartArea.bottom - 10);
              path.lineTo(verticalLineX, chartArea.top);
              ctx.stroke(path);
              ctx.restore();
            }
          },
        },
      ],
    };
  }

  let canvas: HTMLCanvasElement;
  onMount(() => {
    Chart.unregister(...registerables);
    Chart.register([LinearScale, LineController, PointElement, LineElement]);
    const chart = new Chart(
      canvas.getContext('2d') ?? new HTMLCanvasElement(),
      getConfig(),
    );
    return () => {
      chart.destroy();
    };
  });
</script>

<div bind:this={htmlElement} class="h-full w-full relative">
  <div class="z-1 h-full w-full absolute">
    {#if !isNaN(hoverIndex)}
      <p
        style="margin-left: {verticalLineX - 20}px; pointer-events:none;"
        class="absolute mt-20 w-10 text-center">
        {hoverIndex}
      </p>
    {/if}

    <canvas bind:this={canvas} />
  </div>
  <RecordingInspector
    dataPoint={getDataByIndex(hoverIndex)}
    position={modalPosition}
    isOpen={!isNaN(hoverIndex)}
    size={modalSize} />
</div>
