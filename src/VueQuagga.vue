<template>
	<div ref="interactive" id="interactive" class="viewport">
		<video autoplay="true" preload="auto" muted="true" playsinline="true"></video>
		<canvas class="drawingBuffer"></canvas>
	</div>
</template>

<script setup lang="ts">
import Quagga, {
	type QuaggaJSConfigObject,
	type QuaggaJSResultObject,
} from "@ericblade/quagga2";
import { onMounted, onUnmounted, ref, defineExpose } from "vue";

export interface Props {
	debug?: boolean;
	facingMode?: string;
	deviceId?: string;
	patchSize?: string;
	numOfWorkers?: number;
	readers: string[];
}

const props = withDefaults(defineProps<Props>(), {
	debug: false,
	facingMode: "environment",
	patchSize: "medium",
	numOfWorkers: 2,
});

const emit = defineEmits<{
	(e: "barcode-detected", result: QuaggaJSResultObject): void;
}>();

const interactive = ref<null | HTMLDivElement>(null);

const quaggaConfig = ref<QuaggaJSConfigObject>({
	inputStream: {
		type: "LiveStream",
		willReadFrequently: true,
	},
	locator: {
		halfSample: false,
		patchSize: props.patchSize,
	},
	numOfWorkers: self.navigator.hardwareConcurrency,
	decoder: {
		readers: props.readers,
		debug: {
			drawBoundingBox: props.debug,
			showFrequency: props.debug,
			drawScanline: props.debug,
			showPattern: props.debug,
		},
		multiple: false,
	},
	locate: true,
	debug: props.debug,
});

onMounted(() => {
	console.log("H + W:", interactive.value?.offsetHeight, interactive.value?.offsetWidth)
	quaggaConfig.value.inputStream!.constraints = {
		height: interactive.value?.offsetHeight,
		width: interactive.value?.offsetWidth,
		facingMode: props.facingMode,
		deviceId: props.deviceId,
		aspectRatio:
			interactive.value!.offsetWidth / interactive.value!.offsetHeight,
	};

	Quagga.init(quaggaConfig.value, function (err) {
		if (err) {
			return console.error(err);
		}
		Quagga.start();
	});

	Quagga.onDetected(onDetected);
	Quagga.onProcessed(onProcessed);
});

onUnmounted(() => {
	Quagga.offDetected(onDetected);
	Quagga.stop();
});

const onPause = () => {
	console.log("onPause");
	Quagga.offProcessed(onProcessed);
	Quagga.offDetected(onDetected);
};

const onPlay = () => {
	console.log("onPplay");
	Quagga.onDetected(onDetected);
	Quagga.onProcessed(onProcessed);
};

defineExpose({
	onPause,
	onPlay,

});

const onProcessed = (result: QuaggaJSResultObject) => {
	var drawingCtx = Quagga.canvas.ctx.overlay,
		drawingCanvas = Quagga.canvas.dom.overlay;

	if (result) {
		if (result.boxes) {
			drawingCtx.clearRect(
				0,
				0,
				parseInt(drawingCanvas.getAttribute("width") || "0"),
				parseInt(drawingCanvas.getAttribute("height") || "0")
			);
			result.boxes
				.filter(function (box) {
					return box !== result.box;
				})
				.forEach(function (box) {
					Quagga.ImageDebug.drawPath(
						box,
						{ x: 0, y: 1 },
						drawingCtx,
						{
							color: "green",
							lineWidth: 2,
						}
					);
				});
		}

		if (result.box) {
			Quagga.ImageDebug.drawPath(result.box, { x: 0, y: 1 }, drawingCtx, {
				color: "#00F",
				lineWidth: 2,
			});
		}

		if (result.codeResult && result.codeResult.code) {
			Quagga.ImageDebug.drawPath(
				result.line,
				{ x: "x", y: "y" },
				drawingCtx,
				{
					color: "red",
					lineWidth: 3,
				}
			);
		}
	}
};

const onDetected = (result: QuaggaJSResultObject) => {
	if (props.debug) {
		console.log("barcode-detected", result);
	}
	emit("barcode-detected", result);
};
</script>

<style scoped>
#interactive {
	position: relative;
	text-align: center;
}

#interactive video {
	margin: 0 auto;
}

#interactive canvas.drawingBuffer {
	position: absolute;
	left: 0;
	top: 0;
	right: 0;
	bottom: 0;
	margin: 0 auto;
}
</style>
