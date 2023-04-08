<script lang="ts">
	import Konva from 'konva';
	import Stage from '$lib/konva/ResponsiveStage.svelte';
	import type { KonvaEventObject } from 'konva/lib/Node';
	import { getRealPointerPos } from '$lib/konva/util';
	// svelte-konva components
	import Layer from 'svelte-konva/Layer.svelte';
	import Line from 'svelte-konva/Line.svelte';
    import Imagek from 'svelte-konva/Image.svelte';

	import type { LineCap, LineJoin } from 'konva/lib/Shape';
    import { browser } from '$app/environment';
    import { onMount } from 'svelte';
	// import ColorPicker from 'svelte-awesome-color-picker';
	// import {stack} from './../Store'
	// import { history, state } from '../Store';
// import { immerStore, History } from 'svelte-immer-store';
// 	import {noop} from "svelte/internal";
	
// 	const history = new History();
// 	const state = immerStore({
// 		strokes: [],
// 		imgGens: [],
// 		imgUploads: [],
// 		phase: "home",
//     imgsrc: ""
// 	}, noop, history.enqueue);
	
// 	const strokes = state.select(state => state.strokes);
// 	const imgGens = state.select(state => state.imgGens);
// 	const imgUploads = state.select(state => state.imgUploads);
// 	const phase = state.select(state => state.phase);
//   const imgsrc = state.select(state => state.imgsrc);
// 	const {canUndo, canRedo, index, count} = history;

	// export let stage: Konva.Stage;
	let stage: Konva.Stage;
	let imageLayerHandle: Konva.Layer = new Konva.Layer({});
	let maskLayerHandle: Konva.Layer = new Konva.Layer({});

	// let imageLayerHandle: Konva.Layer
	//  let maskLayerHandle: Konva.Layer

	//  let resultsLayerHandle: Konva.Layer
	//  let sketchLayerHandle: Konva.Layer

	 let sketchStrokes: Array<Konva.LineConfig> =[]

	 let maskStrokes: Array<Konva.LineConfig> = []

	 let showMask:boolean
	 let showSketch:boolean
	 let showInput:boolean
	 let showResults:false

	 let selectedMode:string ="txt2image"
	// export let stateHistory:{strokes:Array<Konva.LineConfig>, imgGens:any[], imgUploads:any[], phase:string, imgsrc:string }[]
	// export let currentState:{strokes:Array<Konva.LineConfig>, imgGens:any[], imgUploads:any[], phase:string, imgsrc:string }

	const DRAW_TIMEOUT_MS = 5;
	enum Tools {
		Pen,
		Eraser,
        Select,
	}
	 let selectedTool:any;
	 let strokeWidth:number[]=[8]
	 let opacity:number[]= [1]
	 let tension:number[]= [.1]

	 let hex:string = "#ff0000"; // or hsv or hex
	 let isDrawing = false; // Flag is active if the user is currently drawing/erasing
	 let urlInputImage:boolean;
	// export let height:number ;
	// export let width:number ;
	let height:number = 320 ;
	let width:number = 320;
	let scale:Number = 1;

	$:strokewidth = strokeWidth[0]

	let imgsrcResult:string =""
	let imgsrcInput:string =""

	// export let imgHistory:{imgurl:string, settings:any, type:string  }[];
	// export let strokes: Array<Konva.LineConfig> // This array stores all pen and eraser strokes that have been made

	// let strokeWidth = 30;
	let drawTimeout: NodeJS.Timeout | null; // Timeout used to limit the pointermove event to not save too much data for the stroke points
	let drawTimeoutRunning = false; // Used to indicate wether the timeout is currently in progress or not
	let nstroke = 0;




	async function saveImage(name:string) {
        var dataURL = stage.toDataURL({ pixelRatio: 1 });
        const response = await fetch(
            dataURL
      );
        const imageBlob = await response.blob()
        const imageBase64 = URL.createObjectURL(imageBlob)
        var a = document.createElement('a');
        a.href = imageBase64;
        a.download = name;
        a.click()
        a.remove()
    }


	async function getImageBlob(name:string) {
        var dataURL = imageLayerHandle.toDataURL({pixelRatio: 1});
		// handle1.toDataURL()

        const response = await fetch(
            dataURL
      );
        const imageBlob = await response.blob()
		return await blobToBase64(imageBlob)
    }

	async function getMaskBlob(name:string) {
        var dataURL = maskLayerHandle.toDataURL({ pixelRatio: 1 });
        const response = await fetch(
            dataURL
      );
        const imageBlob = await response.blob()
        // const imageBase64 = URL.createObjectURL(imageBlob)
        return await blobToBase64(imageBlob)
    }

function blobToBase64(blob) {
  const reader = new FileReader();
  reader.readAsDataURL(blob);
  return new Promise(resolve => {
    reader.onloadend = () => {
      resolve(reader.result);
    };
  });
};
	

    function startDraw() {
		const pointerPos = getRealPointerPos(stage.getPointerPosition()!, stage);
        let lineConfig = {
			points: [pointerPos.x, pointerPos.y, pointerPos.x, pointerPos.y], // Initial position is added twice to make a single click visible as dot (otherwise a single click would result in an invisible dot)
			stroke: hex,
            // name: nstroke.toString(),
            strokeWidth: strokewidth,
			lineCap: 'round' as LineCap,
			lineJoin: 'round' as LineJoin,
			tension: tension[0],
            opacity: opacity[0],
			draggable: false,
			globalCompositeOperation: 'source-over' as GlobalCompositeOperation
        }
        if (selectedTool === Tools.Select) {
            lineConfig['fill'] = hex, //Konva.Util.getRandomColor()
            lineConfig['closed'] = true
            lineConfig['opacity'] = opacity[0]
            lineConfig.stroke = null
        }
	
        nstroke += 1;
		if (selectedTool === Tools.Eraser) {
			lineConfig.globalCompositeOperation = 'destination-out';
		}
        // strokesStore.update((current) => {
		// 	current.push(lineConfig); //current.push({});  //  update uses immer, so you can mutate the input and immer will take care of it
		// 	return current;
		// });
		if (imgsrcInput !== "" ){
			console.log("drawing to mask")
			maskStrokes.push(lineConfig);
			maskStrokes = maskStrokes;
		} else {
		console.log("drawing to sketch")
		sketchStrokes.push(lineConfig);
		sketchStrokes = sketchStrokes;
		}
		isDrawing = true;
	}
	function draw() {
		if (!isDrawing) {
			return;
		}
		if (drawTimeout) {
			if (drawTimeoutRunning) {
				return;
			}
			const pointerPos = getRealPointerPos(stage.getPointerPosition()!, stage);
            // console.log("strokes", strokes)

        //     strokesStore.update((current) => {
        //         let points = current[current.length - 1].points!;
        //         (points as Array<number>).push(pointerPos.x);
		// 	(points as Array<number>).push(pointerPos.y);
        //         current[current.length - 1]["points"] = points
        //     // current.push({}); // update uses immer, so you can mutate the input and immer will take care of it
		// 	// current.push(lineConfig); // update uses immer, so you can mutate the input and immer will take care of it
		// 	return current;
		// });


		if (imgsrcInput !== "" ){
			let points = maskStrokes[maskStrokes.length - 1].points!;
			(points as Array<number>).push(pointerPos.x);
			(points as Array<number>).push(pointerPos.y);
			maskStrokes = maskStrokes;
		} else {
		let points = sketchStrokes[sketchStrokes.length - 1].points!;
			(points as Array<number>).push(pointerPos.x);
			(points as Array<number>).push(pointerPos.y);
			sketchStrokes = sketchStrokes;}
		}
		drawTimeoutRunning = true;
		drawTimeout = setTimeout(() => {
			drawTimeoutRunning = false;
		}, DRAW_TIMEOUT_MS);
	}
	function stopDraw(maskStrokes, sketchStrokes) {
		if (!isDrawing) {
			return;
		}

		isDrawing = false;
		drawTimeout = null;
		drawTimeoutRunning = false;
		// let newStack:any
		// if (imgsrcInput !== "" || selectedMode !=="sketch"){
		// 	console.log("stack current", $stack.current, maskStrokes)
        // 	newStack = structuredClone($stack.current)
        // 	newStack["maskStrokes"] =  structuredClone(maskStrokes)
		// } else {
		// 	console.log("stack current", $stack.current, sketchStrokes)
        // 	newStack = structuredClone($stack.current)
        // 	newStack["sketchStrokes"] =  structuredClone(sketchStrokes)
		// }
        // console.log("newStack", newStack)
        // stack.push(newStack)
	}
	function drawMouseOut(e: CustomEvent<KonvaEventObject<PointerEvent>>) {
		const konvaEvent = e.detail;
		// Check if event target is stage (eg. user clicked on empty part of the stage and not any shape)
		if (konvaEvent.target.getType() !== 'Stage') {
			return;
		}
		stopDraw(maskStrokes, sketchStrokes);
	}

let imgInputConf:any;
let imgResultConf:any;



let regions =[];
let setRegions =[];
let selectRegion =[];
let toggleDrawing:boolean = false;

let selectedId:any;
// export let imgsrc
onMount(async()=>{
    // const response = await fetch(
    //     "https://api.unsplash.com/search/photos?page=1&query=sky&client_id=ALjXVfFLvLnhP7NBaWZfpu8UddWaHRPa0Vjf8Z0NlBM"
    //   );
    // const imageBlob = await response.blob()
    // const imageBase64 = URL.createObjectURL(imageBlob)
    // console.log({imageBase64})

    //   console.log(randomImg)
	let imgInput = new window.Image();
	let imgResult = new window.Image();

    if (browser){
		if ( urlInputImage){
			console.log("fetching img" , imgsrcInput)
			const response = await fetch(imgsrcInput );
        const imageBlob = await response.blob()
        const imageBase64 = URL.createObjectURL(imageBlob)
		imgInput.src = imageBase64

		}else{
			imgInput = new window.Image();
			imgInput.src = imgsrcInput
		}

    //     const response = await fetch(
    //     "https://konvajs.org/assets/yoda.jpg"
    //   );
    //     const imageBlob = await response.blob()
    //     const imageBase64 = URL.createObjectURL(imageBlob)
        // img.src = $stack.current.imgsrc //currentState["imgsrc"] //"https://konvajs.org/assets/yoda.jpg";
        imgInput.src = "https://konvajs.org/assets/yoda.jpg";
		// imgInput.crossOrigin = 'Anonymous';
        imgResult.src = "https://konvajs.org/assets/yoda.jpg";

    imgInputConf = {
            width: width,
            height: height,
            image: imgInput,
			scaleX: scale,
			scaleY: scale,

        };
	imgResultConf = {
            width: width,
            height: height,
            image: imgResult,
			scaleX: scale,
			scaleY: scale,

        };


    }

	imageLayerHandle.zIndex(1)
	maskLayerHandle.zIndex(4)

})

$: {
	if (browser){
    let imgInput = new window.Image();
	imgInput.src = imgsrcInput //$stack.current.imgsrc ;
    imgInputConf = {
            width: width,
            height: height,
            image: imgInput,
			scaleX: scale,
			scaleY: scale,
        };
    }
}

$: {
	if (browser){
    let imgResult = new window.Image();
	imgResult.src = imgsrcResult //$stack.current.imgsrc ;
    imgResultConf = {
            width: width,
            height: height,
            image: imgResult,
			scaleX: scale,
			scaleY: scale,
        };
    }
}


let maskConf = {
            width: width,
            height: height,
			scaleX: 1,
			scaleY: 1,
        };

let stageConf = {
            width: width,
            height: height,
			scaleX: 1,
			scaleY: 1,
        };


</script>

<!-- <div class="flex align-middle justify-around m-2">
	<div class="btn-group">
		<button
			class={selectedTool === Tools.Pen ? 'btn btn-active' : 'btn'}
			on:click={() => (selectedTool = Tools.Pen)}>Pen</button
		>
		<button
			class={selectedTool === Tools.Eraser ? 'btn btn-active' : 'btn'}
			on:click={() => (selectedTool = Tools.Eraser)}>Eraser</button
		>
        <button
			class={selectedTool === Tools.Select ? 'btn btn-active' : 'btn'}
			on:click={() => (selectedTool = Tools.Select)}>Select</button
		>
        <button
        class={ 'btn'}
        on:click={async() => { await saveImage("test.png")}}>Save</button>
        <ColorPicker bind:hex />

	</div>
	<div class="flex flex-col justify-around align-middle">
		<span>Size:</span>
		<input type="range" min="1" max="100" bind:value={strokeWidth} class="range range-sm" />
	</div>
</div> -->
<div class="w-full h-full p-0 m-0 bg-transparent border-none border-transparent">
<Stage
	on:pointerdown={startDraw }
	on:pointermove={draw}
	on:pointerup={()=>{stopDraw(maskStrokes, sketchStrokes)}}
	on:mouseout={drawMouseOut}
	bind:handle={stage}
	config={stageConf}
>	
	{#if showInput && imgsrcInput !== ""}
		<Layer handle={imageLayerHandle}>
			<Imagek bind:config={imgInputConf} />
		</Layer>	
	{/if}


	{#if showMask}
	<Layer handle={maskLayerHandle} config={maskConf}>
		<!-- {#if showMask && maskStrokes} -->
		{#each maskStrokes as stroke}
			<Line config={stroke} />
		{/each}
		<!-- {/if} -->
	</Layer>
	{/if}

    <!-- <Layer>
        {#each regions as region}
            <Line
              config={{
              globalCompositeOperation: "destination-out",
              points: region.points.flatMap(p => [p.x, p.y]) ,
              fill: "black",
              listening: false,
              closed: true }}
            />
            <Line
              config= {{
              name: "region",
              points: region.points.flatMap(p => [p.x, p.y]),
              fill : region.color,
              closed: true,
              opacity: region.id === selectedId ? 1 : 0.8 ,
              }}
              on:click={() => {
                selectedId = region.id;
              }}
            />
            {/each}
    </Layer> -->
</Stage>
</div>
