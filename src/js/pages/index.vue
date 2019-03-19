<template>
    <div>
        <div class="status-bar" :style="{'height': statusBarHeight}"></div>
        <div :style='{height:pageHeight}' class="index">
            <div class="listHeader">
                <text class="listHeaderTxt flag">类型</text>
                <text class="listHeaderTxt startTime">开始时间</text>
                <text class="listHeaderTxt remark">备注</text>
                <text class="listHeaderTxt duration">时长</text>
                <text class="listHeaderTxt fromLastTime">距上次</text>
                <text class="listHeaderTxt del">操作</text>
            </div>
            <list class="list">
                <cell v-for="(item,index) in list">
                    <div :class="[item.flag == '母乳' ? 'breast' : 'feedingBottle','listItem']">
                        <div class="iconW flag">
                            <image class="icon" src='bmlocal://assets/breast.png' v-if="item.flag == '母乳'"></image>
                            <image class="icon" src='bmlocal://assets/feedingBottle.png' v-else></image>
                        </div>
                        <text class="listTxt startTime">{{filterTime(item.startTime)}}</text>
                        <text class="listTxt remark" :class="[item.remark == '左' ? 'breast_left':'',item.remark == '右'?'breast_right':'']">{{item.remark}}</text>
                        <text class="listTxt duration">{{filterDuration(item.duration)}}</text>
                        <text class="listTxt fromLastTime">{{filterDuration(item.fromLastTime)}}</text>
                        <div class="delW del" v-if="showCheckbox">
                            <wxc-checkbox :value='index' @wxcCheckBoxItemChecked="wxcCheckBoxItemChecked" :checked="item.checked"></wxc-checkbox>
                        </div>
                        <div class="delW del" v-else @click='del(index)' @longpress='showCheckbox = true'>
                            <image class="icon-del" src='bmlocal://assets/del.png'></image>
                        </div>
                    </div>
                </cell>
            </list>
            <div class="delAll" v-if="showCheckbox">
                <div class="da_btns">
                    <text class="da_btn da_cancel" @click="showCheckbox = false">取消</text>
                    <text class="da_btn da_confirm" @click="confirmDel">确定删除</text>
                </div>
                <wxc-checkbox title="全选" :checked="false" @wxcCheckBoxItemChecked='checkAll = !checkAll'></wxc-checkbox>
            </div>
            <text class="fromLastTimeLine" v-if="fromLastTime > 0">距上次喂食：{{fromLastTime_com}}</text>
            <div class="btns">
                <div class="left half" v-if="breastState == '开始'" @click="feedingBottleClick">
                    <text class="f-c-w btn-txt">奶瓶 {{milkVol}}ml</text>
                    <div class="setpper">
                        <text @click="changeMilkVol('add')" class="s-btn">+</text>
                        <text @click="changeMilkVol('red')" class="s-btn">-</text>
                    </div>
                </div>
                <div class="right half" @click="breastClick" :class="[breastPosition == '右' ? 'breast_right_bg':'']">
                    <text class="f-c-w btn-txt">母乳-{{breastPosition}}-{{breastState}}</text>
                    <text class="f-c-w btn-txt" v-if="eattingTime > 0">已吃：{{eattingTime_com}}</text>
                    <div class="toggleBreast" @click="toggleBreast" v-if="breastState == '开始'">
                        <image class="toggleBreastIcon" src='bmlocal://assets/toggleBreast.png'></image>
                    </div>
                </div>
            </div>
        </div>
        <div class="mask" v-if="bigVolShow" :style="{height:realDeviceHeight}">
            <div class="bigVal">
                <text class="bigValTxt">{{milkVol}}</text>
            </div>
        </div>

        <wxc-mask height="400"
                    width="702"
                    border-radius="0"
                    duration="200"
                    mask-bg-color="#FFFFFF"
                    :has-animation="true"
                    :has-overlay="true"
                    :show-close="true"
                    :show="showMask"
                    @wxcMaskSetHidden="wxcMaskSetHidden">
            <div class="content">
                <scroller class="s">
                    <text class="u_title">{{updateInfo.updateTime}}更新内容</text>
                    <div class="u_item" v-for="(u_item,i) in updateInfo.updateCon">
                        <text class="u_n">{{i+1}}.</text>
                        <text class="u_t">{{u_item}}</text>
                    </div>
                </scroller>
            </div>
        </wxc-mask>
    </div>
</template>

<script>
let milkVolTimer,fromLastTimer,eattingTimer,h,m,s;
let LIST_MAX_LENGTH = 50;  //列表最大长度
import { WxcCheckbox,WxcMask } from 'weex-ui';
import newUpdate from '../json/updateRecord'
export default {
    components: { WxcCheckbox,WxcMask },
    data () {
        return {
            fromLastTime : 0,   //距离上次吃奶时间
            eattingTime  : 0,   //已吃时间
            list : [],
            milkVol : 0,
            bigVolShow : false,
            breastPosition : '',
            breastState : '',
            curHomeBackTriggerTimes: 0,
            maxHomeBackTriggerTimes: 1,
            navBarHeight: Number(weex.config.eros.navBarHeight),
            statusBarHeight: Number(weex.config.eros.statusBarHeight),
            tabbarHeight: Number(weex.config.eros.tabbarHeight),
            realDeviceHeight: (750 / weex.config.eros.deviceWidth) * weex.config.eros.deviceHeight,
            platform: weex.config.env.platform,
            isiOs: weex.config.env.platform == 'iOS',
            showMask : false,            //弹层
            updateInfo : [],
            showCheckbox : false,
            checkAll : false,
            checkedList : []
        };
    },
    created () {
        let breastPosition = this.$storage.getSync('breastPosition')
        let breastState = this.$storage.getSync('breastState')
        let milkVol = this.$storage.getSync('milkVol')
        let list = this.$storage.getSync('list')
        this.list = Array.isArray(list) ? list : []
        this.breastPosition = breastPosition ? breastPosition : '左'
        this.breastState = breastState ? breastState : '开始'
        this.milkVol = milkVol ? milkVol : 50
        this.comEattingTime()
        this.comTimeFromLast()
        this.androidFinishApp()
        this.checkUpdate()
    },
    eros : {
        appActive(){
            this.comTimeFromLast()
            this.comEattingTime()
        }
    },
    methods: {
        confirmDel(){
            let newList = this.list.filter(function(item){
                return !item.checked
            })
            this.list.splice(0)
            this.list.push(...newList)
            this.$storage.set('list', this.list)
            this.showCheckbox = false
        },
        wxcCheckBoxItemChecked(e){
            let i = e.value
            let item = this.list[i]
            item.checked = e.checked
            this.list.splice(i,1,item)
        },
        checkUpdate(){
            let updateReadState = this.$storage.getSync('updateReadState')
            if (!updateReadState) {
                this.updateInfo = newUpdate
                this.showMask = true
            }
        },
        wxcMaskSetHidden () {
            this.showMask = false;
            this.$storage.set('updateReadState', true)
                .then(resData => {}, error => {})
        },
        changeMilkVol(f){
            clearTimeout(milkVolTimer)
            if (f == 'add') {
                this.milkVol += 5
            }else if (f == 'red') {
                this.milkVol -=5
            }
            this.bigVolShow = true
            milkVolTimer = setTimeout(() => {
                this.bigVolShow = false
            }, 1000);
        },
        comTimeFromLast(){
            //计算距离上一次喂食过去多长时间
            if (this.list.length > 0) {
                clearInterval(fromLastTimer)
                let d = new Date()
                let t = d.getTime()
                let lastItem = this.list[0]
                let fromLastTime
                if (lastItem.flag == '母乳' && lastItem.endTime != '') {
                    fromLastTime = t - lastItem.endTime
                }else if(lastItem.flag == '奶瓶'){
                    fromLastTime = t - lastItem.startTime
                }else{
                    return false
                }
                this.fromLastTime = Math.floor(fromLastTime / 1000)
                fromLastTimer = setInterval(() => {
                    this.fromLastTime++
                }, 1000);
            }
        },
        comEattingTime(){
            if (this.breastState == '结束') {
                clearInterval(eattingTimer)
                let d = new Date()
                let t = d.getTime()
                let lastItem = this.list[0]
                this.eattingTime = Math.floor((t - lastItem.startTime)/1000)
                eattingTimer = setInterval(() => {
                    this.eattingTime++
                }, 1000);
            }
        },
        filterTime(date){
            if (date != '') {
                let unixTimestamp = new Date(date) ;
                let t = add0(unixTimestamp.getHours())+ ':' + add0(unixTimestamp.getMinutes())
                function add0(date){
                    if (date < 10) {
                        return '0' + date
                    }else{
                        return date
                    }
                }
                return t
            }
        },
        filterDuration(min){
            let m = Math.floor(min)
            if (min != '') {
                if (Number(m) >= 60) {
                    return '' + Math.floor(m/60) + 'h ' + (m%60).toString() + 'm'
                }else{
                    return Math.floor(m) + 'm'
                }
            }
        },
        del(index){
            let _this = this
            this.$notice.confirm({
                title: '警告',
                message: '是否确认删除此条记录',
                okTitle: '确认',
                cancelTitle: '取消',
                okCallback() {
                    _this.list.splice(index,1)
                    _this.$storage.set('list', _this.list)
                },
                cancelCallback() {
                    // 点击取消按钮的回调
                }

            })
        },
        feedingBottleClick(){
            let startTime = new Date()
            let fromLastTime = 0
            if (this.list.length > 0) {
                let lastItem = this.list[0]
                if (lastItem.flag == '母乳') {
                    fromLastTime = startTime - lastItem.endTime
                }else{
                    fromLastTime = startTime - lastItem.startTime
                }
            }
            this.list.unshift({
                flag : '奶瓶',
                startTime : startTime.getTime(),
                endTime : '',
                duration : '',
                remark : this.milkVol + 'ml',
                fromLastTime :  fromLastTime/1000/60,
                checked : false
            })
            if (this.list.length > LIST_MAX_LENGTH) {
                this.list.splice(LIST_MAX_LENGTH,this.list.length-LIST_MAX_LENGTH)
            }
            this.$storage.set('milkVol', this.milkVol)
            this.$storage.set('list', this.list)
            this.comTimeFromLast()
        },
        breastStart(){
            clearInterval(fromLastTimer)
            this.fromLastTime = 0
            let startTime = new Date()
            let fromLastTime = 0
            if (this.list.length > 0) {
                let lastItem = this.list[0]
                if (lastItem.flag == '母乳') {
                    fromLastTime = (startTime - lastItem.endTime)/1000/60
                }else{
                    fromLastTime = (startTime - lastItem.startTime)/1000/60
                }
            }
            this.list.unshift({
                flag : '母乳',
                startTime : startTime.getTime(),
                endTime   : '',
                duration  : '',            //持续时长
                remark    : this.breastPosition,
                fromLastTime : fromLastTime,
                checked : false
            })
            if (this.list.length > LIST_MAX_LENGTH) {
                this.list.splice(LIST_MAX_LENGTH,this.list.length-LIST_MAX_LENGTH)
            }
            this.$storage.set('list', this.list)
                .then(resData => {
                    console.log('ok')
                    this.breastState = '结束'
                    this.$storage.setSync('breastPosition', this.breastPosition)
                    this.$storage.setSync('breastState', '结束')
                    this.comEattingTime()
                }, error => {})
        },
        breastEnd(){
            clearInterval(eattingTimer)
            this.eattingTime = 0
            let lastItem = this.list[0]
            if (lastItem.flag == '母乳' && lastItem.endTime == '') {
                let endTime = new Date()
                let duration = (endTime - lastItem.startTime)/1000/60
                lastItem.endTime = endTime.getTime()
                lastItem.duration = duration
                this.list.splice(0,1,lastItem)
                this.$storage.set('list', this.list)
                this.breastPosition = this.breastPosition == '左' ? '右' : '左'
                this.$storage.setSync('breastPosition', this.breastPosition)
                this.$storage.setSync('breastState', '开始')
            }else{
                let _this = this
                this.$notice.alert({
                    title: '警告',
                    message: '发生错误，记录失败',
                    okTitle: '确认',
                    callback() {
                        _this.$storage.setSync('breastState', '开始')
                    }
                })
            }
            this.breastState = '开始'
            this.comTimeFromLast()
        },
        breastClick(){
            if (this.breastState == '开始') {
                this.breastStart()
            }else{
                this.breastEnd()
            }
        },
        toggleBreast(){
            if (this.breastState == '开始') {
                this.breastPosition = this.breastPosition == '左' ? '右' : '左'
            }
        },
        androidFinishApp () {
            const globalEvent = weex.requireModule('globalEvent')
            globalEvent.addEventListener('homeBack', options => {
                (this.curHomeBackTriggerTimes === this.maxHomeBackTriggerTimes) && this.$router.finish()
                this.curHomeBackTriggerTimes++
                this.$notice.toast({ message: `再次点击，关闭应用` })
            })
        }
    },
    computed: {
        fromLastTime_com(){
            if (this.fromLastTime < 60) {
                return this.fromLastTime + '秒'
            }else if (this.fromLastTime >= 60) {
                if (this.fromLastTime < 3600) {
                    return Math.floor(this.fromLastTime / 60) + '分 ' + this.fromLastTime%60 + '秒'
                }else if (this.fromLastTime >= 3600) {
                    h = Math.floor(this.fromLastTime / 3600)
                    m = Math.floor((this.fromLastTime - h*3600)/60)
                    s = this.fromLastTime - h*3600 - m*60
                    return h + '小时 ' + m + '分 ' + s + '秒'
                }
            }
        },
        eattingTime_com(){
            return Math.floor(this.eattingTime/60) + '分 ' + this.eattingTime%60 + '秒'
        },
		pageHeight() {
			if (this.platform == 'iOS') {
				return Number(this.realDeviceHeight - this.statusBarHeight)
			}else{
				return Number(weex.config.eros.realDeviceHeight - this.statusBarHeight)
			}
		}
    },
    watch : {
        checkAll(val){
            let delItem;
            for (let i = 0; i < this.list.length; i++) {
                delItem = this.list[i]
                delItem.checked = val
                this.list.splice(i,1,delItem)
            }
        },
        list(val){
            if (val.length == 0) {
                clearInterval(fromLastTimer)
                this.fromLastTime = 0
            }
        }
    }
}
</script>
<style lang="scss" src='./index.scss'></style>