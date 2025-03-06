<template>
	<view class="water-fall-container" ref='outerContainer'>
		<view class="container" ref="container">
		</view>
		<text class="loading...">加载中</text>
	</view>
</template>

<script>
import { debounce, throttle, measureTextWidth } from '../../../utils';
const PAGE_SIZE = 20;
const TEXT_PADDING = 12;//文字部分的内边距
const TEXT_LINEHEIGHT = 24;//文字高度
const TEXT_FONT = '16px sans-serif';
const COLUMN_GAP = 20;
const CONTAINER_PADDING_BOTTOM = 20;//容器底部空间
const SCROLL_DOWN = 'down';
const SCROLL_UP = 'up';
const VISIBLE_PADDING = 400;//在视窗外部上下VISIBLE_PADDING间距内的元素也渲染
// interface ImgPosition {
// 	img: string;//图片地址
// 	idx: number;//图片索引
// 	columnIdx: number;//图片所在列的索引
// 	left: number;//图片左边距容器左边框的距离
// 	top: number;//图片顶部距容器顶部距离
// 	width: number;//图片宽度
// 	height: number;//图片高度
// 	lineHeight: number;//文字高度（todo）
// }

export default {
    props: {
        getImageList: {
            type: Function,
            required: true,
        }
    },
    data() {
        return {
            imageList: [],
            imagePositionInfo: [],//图片位置信息array<ImgPosition>
            startIdx: 0,//已经渲染的图片的起始index
            endIdx: 0,//已经渲染的图片的结束index
            columnNum: 2,//列的数量
            containerWidth: 0,
            columnList: [],//array<{columnIdx: number, columnHeight: number}>
            renderedList: {},//已渲染的图片
            lastScrollTop: 0,//滚动前的scrollTop
            isLoadNextPage: false,
            hasNextPage: true,
            resizeCb: null,
        }
    },
    computed: {
        columnWidth() {
            if(this.containerWidth === 0) {
                return 0;
            }
            const allGapSize = COLUMN_GAP * (this.columnNum - 1);
            return (this.containerWidth - allGapSize)/this.columnNum
        },
        containerEl() {
            return this.$refs.container?.$el;
        },
        outerContainerEl() {
            return this.$refs.outerContainer?.$el;
        },
    },
    onLoad() {
    },
    async mounted() {
        this.resizeEventRegistry();
        this.scrollEventRegistry();
        this.columnNum = this.getColumnNum();
        this.columnList = this.getColumnList();
        //获得列数之后，再开始请求图片列表，计算图片尺寸
        const {newList, start} = await this.fetchImageList();
        this.updateImagePositionInfo(newList, start);
        this.setContainerHeight();
        this.renderImage(start);
    },
    methods: {
        async fetchImageList() {
            this.isLoadNextPage = true;
            const start = this.imageList.length;
            const ret = await this.getImageList(PAGE_SIZE);
            this.isLoadNextPage = false;
            this.hasNextPage = ret.length === PAGE_SIZE;
            this.imageList = this.imageList.concat(ret);
            return {newList: ret, start }
        },
        getColumnNum() {
            if(!this.containerEl) {
                return;
            }
            const boxWidth = this.containerWidth = this.containerEl.offsetWidth;
            if (boxWidth >= 1600) return 5;
                else if (boxWidth >= 1200) return 4;
                else if (boxWidth >= 768 && boxWidth < 1200) return 3;
                else return 2;
        },
        getColumnList(){
            const {columnNum} = this;
            return Array(columnNum).fill(0).map((v,i)=> ({
                columnIdx: i,
                columnHeight: 0,
                left: (COLUMN_GAP + this.columnWidth) * i,
            }))
        },
        //更新图片位置表
        updateImagePositionInfo(list, startRenderIdx = 0) {
            if(this.columnWidth === 0) {
                return []
            }
            const tempListInfo = [];
            for(let i = 0; i < list.length; i ++) {
                //把最短的列排到最前面
                this.columnList.sort((a,b) => {
                    return a.columnHeight - b.columnHeight;
                });
                //当前图片放在最短的列上
                const params = {
                    ...list[i],
                    idx: startRenderIdx + i,
                    columnIdx: this.columnList[0].columnIdx,
                    left: this.columnList[0].left,
                    top: this.columnList[0].columnHeight,
                    width: this.columnWidth,
                    imgHeight: list[i].height * (this.columnWidth / list[i].width),
                    textHeight: this.getTextHeight(list[i].text),
                }
                params.height = params.imgHeight + params.textHeight + 10;
                this.columnList[0].columnHeight += params.height + COLUMN_GAP;
                tempListInfo.push(params);
            }
            this.imagePositionInfo = this.imagePositionInfo.concat(tempListInfo)
        },
        getTextHeight(text) {
            const width = measureTextWidth(text, TEXT_FONT);
            const textContainerWidth = this.columnWidth - TEXT_PADDING * 2;
            const mul = Math.ceil(width / textContainerWidth);
            console.log(width, textContainerWidth, mul)
            console.log(mul * TEXT_LINEHEIGHT, TEXT_PADDING * 2)
            return mul * TEXT_LINEHEIGHT + TEXT_PADDING * 2
        },
        setContainerHeight() {
            this.columnList.sort((a,b) => a.columnHeight - b.columnHeight);
            if(this.containerEl) {
                this.containerEl.style.height = this.columnList[this.columnList.length - 1].columnHeight + CONTAINER_PADDING_BOTTOM + 'px';
            }
        },
        renderImage(startBy = 0) {
            if(this.imagePositionInfo.length === 0 || !this.containerEl) {
                return;
            }
            const tempRenderMap = {};
            for(let i = startBy; i < this.imagePositionInfo.length; i ++) {
                const { idx } = this.imagePositionInfo[i];
                //检查当前元素是否在视窗中可见
                const { overTopLine, underBottomLine } = this.checkVisible(this.imagePositionInfo[i]);
                const dom = this.containerEl.querySelector(`.item_${idx}`)
                if(overTopLine || underBottomLine) {
                    //不应该渲染
                    dom?.remove();
                    continue;
                }
                if(dom) {
                    tempRenderMap[idx] = this.createDom(dom, this.imagePositionInfo[i]);
                } else {
                    tempRenderMap[idx] = this.createDom(document.createElement('div'), this.imagePositionInfo[i]);
                    this.containerEl.appendChild(tempRenderMap[idx])
                }
                // 对象的key是无序的，但数值键会自动排序，所以keys的首项和末项，就是最小最大值
                const keys = Object.keys(Object.assign(this.renderedList, tempRenderMap));
                this.startIdx = +keys[0];
                this.endIdx = +keys[keys.length - 1];
            }
        },
        createDom(domNode, info) {
            let imgEl = domNode.querySelector('img');
            let textEl = domNode.querySelector('p');
            if(!imgEl) {
                imgEl = document.createElement('img');
                domNode.appendChild(imgEl);
            }
            if (!textEl) {
                textEl = document.createElement('p');
                domNode.appendChild(textEl);
            }
            const { img, width, height, imgHeight, textHeight, left, top, idx, text } = info;
            imgEl.src = img;
            imgEl.style.width = width + 'px';
            imgEl.style.height = imgHeight + 'px';
            textEl.style.padding = TEXT_PADDING + 'px';
            textEl.style.height = textHeight + 'px';
            textEl.style.lineHeight = TEXT_LINEHEIGHT + 'px';
            textEl.style.boxSizing = 'border-box';
            textEl.style.font = TEXT_FONT;
            textEl.innerHTML = text;
            domNode.style.width = width + 'px';
            domNode.style.height = height + 'px';
            domNode.style.transform = `translate(${left}px, ${top}px)`;
            // 样式没有编译成功，设置style
            domNode.style.display = 'flow-root';
            domNode.style.position = 'absolute';
            domNode.style.left = '0';
            domNode.style.top = '0';
            domNode.style.boxShadow = '1px 2px 8px #eee';
            
            domNode.setAttribute('class', `item_${idx} list_item`)
            return domNode;
        },
        //检查元素是否可见
        checkVisible(imgInfo) {
            if(!this.outerContainerEl) {
                return {}
            }
            const topLine = this.outerContainerEl.scrollTop;
            const bottomLine = this.outerContainerEl.scrollTop + this.outerContainerEl.offsetHeight;
            const { top, height } = imgInfo;
            const imgBottom = top + height;
            if (imgBottom + VISIBLE_PADDING < topLine) {
                return {
                    //在视窗上部，不可见
                    overTopLine: true
                }
            } else if(top - VISIBLE_PADDING > bottomLine) {
                return {
                    //在视窗下部，不可见
                    underBottomLine: true
                }
            } else {
                return {}
            }
        },
        updateDomPosition(direction) {
            const tempRenderMap = {};
            //检查已渲染列表中的元素，不符合条件删除元素，反之插入元素
            for(let i = this.startIdx; i <= this.endIdx; i++){
                const { overTopLine, underBottomLine } = this.checkVisible(this.imagePositionInfo[i]);
                const dom = this.containerEl?.querySelector(`.item_${i}`);
                if(overTopLine || underBottomLine) {
                    dom?.remove();
                } else if(this.renderedList[i]) {
                    tempRenderMap[i] = this.renderedList[i]
                } else {
                    tempRenderMap[i] = this.createDom(document.createElement('div'), this.imagePositionInfo[i]);
                    this.containerEl.appendChild(tempRenderMap[i])
                }
            }
            if(direction === SCROLL_DOWN) {
                for(let i = this.endIdx + 1; i < this.imagePositionInfo.length; i ++) {
                    const { underBottomLine } = this.checkVisible(this.imagePositionInfo[i]);
                    //向下找到第一个在视窗下部的元素，停止渲染
                    if(underBottomLine) {
                        break;
                    }
                    const dom = this.containerEl?.querySelector(`.item_${i}`);
                    if(dom) {
                        tempRenderMap[i] = this.imagePositionInfo[i];
                    } else {
                        tempRenderMap[i] = this.createDom(document.createElement('div'), this.imagePositionInfo[i]);
                        this.containerEl.appendChild(tempRenderMap[i]);
                    }
                }
            } else if(direction === SCROLL_UP) {
                for(let i = this.startIdx - 1; i >= 0; i --) {
                    const { overTopLine } = this.checkVisible(this.imagePositionInfo[i]);
                    //向上找到第一个在视窗上部的元素，停止渲染
                    if(overTopLine) {
                        break;
                    }
                    const dom = this.containerEl?.querySelector(`.item_${i}`);
                    if(dom) {
                        tempRenderMap[i] = this.imagePositionInfo[i];
                    } else {
                        tempRenderMap[i] = this.createDom(document.createElement('div'), this.imagePositionInfo[i]);
                        this.containerEl.appendChild(tempRenderMap[i]);
                    }
                }
            }
            this.renderedList = tempRenderMap;
            const keys = Object.keys(this.renderedList);
            this.startIdx = +keys[0];
            this.endIdx = +keys[keys.length - 1];
        },
        resizeEventRegistry() {
            const resizeFn = () => {
                this.columnNum = this.getColumnNum();
                this.columnList = this.getColumnList();
                this.renderedList = {};
                this.imagePositionInfo = [];
                this.startIdx = this.endIdx = 0;
                this.updateImagePositionInfo(this.imageList);
                this.setContainerHeight();
                this.renderImage(0);
            }
            const resize = debounce(() => {
                if(this.isLoadNextPage) {
                    //加载数据中，发生了宽度变化，缓存回调，数据加载完后调用
                    this.resizeCb = resizeFn;
                    return
                }
                resizeFn();
            }, 150)
            globalThis.addEventListener("resize", resize);
        },
        scrollEventRegistry() {
            const container = this.$refs.outerContainer?.$el;
            if(!container) {
                return;
            }
            const handleScroll = throttle(async () => {
                const curScrollTop = container.scrollTop;
                const scrollDirection = curScrollTop > this.lastScrollTop ? SCROLL_DOWN : SCROLL_UP;
                this.lastScrollTop = curScrollTop;
                //根据滚动方向，处理元素显隐
                this.updateDomPosition(scrollDirection);
                if(this.isLoadNextPage || !this.hasNextPage) {
                    return false;
                }
                if(curScrollTop + container.offsetHeight >= container.scrollHeight * 0.85) {
                    //当滚动到底部还有15%的高度时，请求下一页
                    const { newList, start } = await this.fetchImageList();
                    // 请求数据期间发生了宽度变化
                    if(this.resizeCb) {
                        //重新处理列数据，并渲染图片
                        this.resizeCb && this.resizeCb();
                        this.resizeCb = null;
                    } else {
                        // 处理新数据
                        this.updateImagePositionInfo(newList, start);
                        this.setContainerHeight();
                        this.renderImage(start);
                    }
                }
                
            }, 150)
            container.addEventListener('scroll', handleScroll)
        },
    }
}
</script>

<style scoped>
.water-fall-container {
	height: calc(100vh - 44px);
	overflow-y: scroll;
}
.container {
	position: relative;
	width: 100%;
}
.list_item {
    position: absolute;
    left: 0;
    top: 0;
    display: flow-root;
}

</style>
