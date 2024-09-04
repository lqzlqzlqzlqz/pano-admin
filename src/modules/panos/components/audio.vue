<template>
	<div class="audio-card">
		<audio ref="audioRef" controls="controls" :disabled="!modelValue">
			<source :src="musicUrl" autoPlay />
			Your browser does not support the audio element.
		</audio>
	</div>
	<cl-upload type="file" text="设置背景音乐" v-model="musicUrl" @change="musicChange" />
</template>

<script lang="ts" setup>
import { ref } from "vue";
defineProps<{
	modelValue: string;
}>();
const emit = defineEmits(["update:modelValue"]);
const musicUrl = ref("");
function musicChange() {
	emit("update:modelValue", musicUrl.value);
}
</script>

<style>
.audio-card {
}
</style>