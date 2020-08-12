<template>
  <div class="menu">
    <van-swipe indicator-color="#ffffff" style="height: 163.03px;">
      <van-swipe-item v-for="(item, index) in bannerData" :key="index">
        <img class="auto-img" v-lazy="item.bannerImg" alt />
      </van-swipe-item>
    </van-swipe>

    <div class="menu-content">
      <div class="aside-menu fl">
        <div class="item" :ref="'item' + index" :class="{active: item.isActive}" v-for="(item, index) in typeData" :key="index" @click="toggleAsideMenu(item, index)">{{item.typeDesc}}</div>
        <div class="line" :style="{top: top + 'px'}"></div>
      </div>
      <div class="content fl clearfix">
        <div class="fl content-item" @click="goDetail(item.pid)" :class="[index % 2 == 0 ? 'left' : 'right']" v-for="(item, index) in productData" :key="index">
          <div>
            <img
              class="auto-img"
              v-lazy="item.smallImg"
              alt
            />
          </div>
          <div class="pro-name one-text">{{item.name}}</div>
          <div class="pro-enname one-text">{{item.enname}}</div>
          <div class="pro-price">￥{{item.price}}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import '../../assets/css/menu.less'
import {createNamespacedHelpers} from 'vuex'

const {mapState, mapMutations} = createNamespacedHelpers('menuModule')

export default {
  data() {
    return {
      //banner图
      bannerData: [],

      //商品类型
      typeData: [],

      //移动线条的top
      top: 0,

      //侧边栏菜单的标签高度
      // height: 0,

      //商品数据
      productData: []
    };
  },

  created() {
    //获取banner数据
    this.getBannerData();

    //获取商品类型数据
    this.getTypeData();

    //根据类型获取商品数据
    this.getProductByType(this.productType);

    this.moveLine();
  },

  computed: {
    ...mapState(['bannerDataCache', 'typeDataCache', 'products', 'height', 'productType'])
  },

  methods: {

    ...mapMutations(['changeData', 'changeProducts']),

    //初始化时移动线条
    moveLine() {
      
      

      for (let i = 0; i < this.typeData.length; i++) {
        if (this.typeData[i].isActive) {
          this.top = i * this.height;
          return;
        }
      }
      
    },

    //获取banner数据
    getBannerData() {

      //获取vuex的banner缓存数据，如果存在缓存数据，则不发起请求，否则发起请求
      if (this.bannerDataCache.length > 0) {
        this.bannerData = this.bannerDataCache;
        
        return;
      }


      //开启加载提示
      this.$toast.loading({
        message: "加载中...",
        forbidClick: true,
        //延迟自动关闭加载提示，如果该值为0，则不会自动关闭
        duration: 0
      });

      this.axios({
        method: 'GET',
        url: '/banner',
        params: {
          appkey: this.appkey
        }
      }).then(result => {
        this.$toast.clear();
        
        if (result.data.code == 300) {
          this.bannerData = result.data.result;
          

          //缓存banner数据在vuex中
          // this.$store.commit('menuModule/changeData', {
          //   key: 'bannerDataCache',
          //   value: result.data.result
          // })

          this.changeData({key: 'bannerDataCache', value: result.data.result});

          
        }
      }).catch(err => {
        this.$toast.clear();
        
      })
    },

    //获取商品类型数据
    getTypeData() {

      //获取vuex的侧边栏菜单缓存数据，如果存在缓存数据，则不发起请求，否则发起请求
      if (this.typeDataCache.length > 0) {
        this.typeData = this.typeDataCache;
        
        return;
      }

      //开启加载提示
      this.$toast.loading({
        message: "加载中...",
        forbidClick: true,
        //延迟自动关闭加载提示，如果该值为0，则不会自动关闭
        duration: 0
      });
      this.axios({
        method: 'GET',
        url: '/type',
        params: {
          appkey: this.appkey
        }
      }).then(result => {
        this.$toast.clear();
        
        if (result.data.code == 400) {
          result.data.result.forEach(v => {
            v.isActive = false;
          })
          result.data.result.unshift({type: 'isHot', typeDesc: '最新推荐', isActive: true});
          this.typeData = result.data.result;

          this.$nextTick(() => {
            //获取侧边栏菜单标签高度
            let firstItemHeight = this.$refs.item0[0].clientHeight;
            // this.height = firstItemHeight;
            this.changeData({
              key: 'height',
              value: firstItemHeight
            })
          })

          //缓存侧边栏菜单数据在vuex
          this.changeData({
            key: 'typeDataCache',
            value: result.data.result
          })
          

        }
        
      }).catch(err => {
        this.$toast.clear();
        
      })
    },

    //切换标签
    toggleAsideMenu(item, index) {
      if (item.isActive) {
        return;
      }

      for (let i = 0; i < this.typeData.length; i++) {
        if (this.typeData[i].isActive) {
          this.typeData[i].isActive = false;
          break;
        }
      }

      item.isActive = true;

      //移动线条
      this.top = this.height * index;
      
      //记录当选选择的商品类型，主要纠正查看详情，再返回菜单，导致菜单数据显示不正确问题
      this.changeData({
        key: 'productType',
        value: {
          type: item.type
        }
      })

      //根据类型获取商品数据
      this.getProductByType(item);

    },

    //根据类型获取商品数据
    getProductByType(item) {

      

      //在缓存获取商品数据
      if (this.products[item.type]) {
        this.productData = this.products[item.type];
        
        return;
      }

      this.productData = [];

      let params = {
        appkey: this.appkey,
        key: 'type',
        value: item.type
      };

      //最近推荐
      if (item.type == 'isHot') {
        params.key = 'isHot';
        params.value = 1;
      }

      //开启加载提示
      this.$toast.loading({
        message: "加载中...",
        forbidClick: true,
        //延迟自动关闭加载提示，如果该值为0，则不会自动关闭
        duration: 0
      });

      this.axios({
        method: 'GET',
        url: '/typeProducts',
        params
      }).then(result => {
        this.$toast.clear();
        
        if (result.data.code == 500) {
          this.productData = result.data.result;
          
          //缓存商品数据
          this.changeProducts({
            type: item.type,
            value: result.data.result
          })
          
          
        }
      }).catch(err => {
        this.$toast.clear();
        
      })
      
    },

    goDetail(pid) {
      //pid: 商品id
      this.$router.push({name: 'Detail', query: {pid}});
    }
  }
};
</script>

<style lang="less" scoped>

</style>