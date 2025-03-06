<template>
	<WaterFallList :getImageList="getImageList"/>
</template>

<script>
	import WaterFallList from './components/WaterfallVirtual.vue';
	import img10x30 from '../../static/100*300.jpg';
	import img20x20 from '../../static/200*200.jpg';
	import img20x46 from '../../static/200*460.jpg';
	import img24x10 from '../../static/240*100.jpg';
	import img24x20 from '../../static/240*200.jpg';
	
	const IMAGE_ARR = [
		{
			img: img10x30,
			width: 100,
			height: 300,
		}, 
		{
			img: img20x20,
			width: 200,
			height: 200,
		}, 
		{
			img: img20x46,
			width: 200,
			height: 460,
		}, 
		{
			img: img24x10,
			width: 240,
			height: 100,
		}, 
		{
			img: img24x20,
			width: 240,
			height: 200
		}
	];
	export default {
		components: {
			WaterFallList,
		},
		data() {
			return {
			}
		},
		onLoad() {
		},
		mounted() {
		},
		methods: {
			async getImageList(pageSize) {
				return new Promise(resolve => {
					const ret = [];
					for(let i = 0; i < pageSize; i++) {
						const imgIdx = Math.floor(Math.random() * 5);
						ret.push({...IMAGE_ARR[imgIdx]})
					}
					setTimeout(() => {
						resolve(ret);
					}, 1000)
				})
			},
		}
	}
</script>

<style>
</style>
