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

const TEST_LIST = [
  "我们...还不太熟吧",
  "用一场远行，当做逃离的方舟；做生活困兽，不如去浪漫宇宙",
  "找遍了记忆商店，他说想再荡个秋千，妈妈我想飞的高点！星球以光速穿越，妈妈我想落到地面",
  "在电幻的荒丘 寻真实的绿洲；渺小得如蜉蝣 也仰望着宇宙；那来自过去 古老的眼神；如何能辨认 此刻是幻是真；人造的天分 是否算慧根；克隆的肉身 是否有灵魂",
  "冬天来得很迟；你有我的地址；信箱里有钥匙 没人知；你是我的眼泪；琴弦上的玫瑰；天亮时的派对 好滋味"
];
	
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
						const idx = Math.floor(Math.random() * 5);
						ret.push({...IMAGE_ARR[idx], text: TEST_LIST[idx]})
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
