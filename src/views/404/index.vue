<template>
  <div id="notFound" class="d-flex flex-jcc" v-once>
    <div class="not-found-content d-flex flex-clo flex-ai">
      <img class="not-found-img" :src="`${qiNiuImgLink}404.svg`" alt="">
      <span class="title-xl">哦。。。页面哪儿去了？</span>
      <router-link to="/">
        <v-btn color="primary" class="go-home-btn" dark>返回首页</v-btn>
      </router-link>
    </div>
  </div>
</template>

<script>
import { qiNiuImgLink } from '@utils/publicData'
export default {
  data() {
    return {
      qiNiuImgLink,
    }
  },
}
</script>

<style lang="scss" scoped>
#notFound {
  .not-found-content {
    .go-home-btn {
      margin-top: 25px;
      width: 250px;
    }
  }
}

@include screenLG {
  #notFound {
    min-height: calc(100vh - 70px);
    .not-found-content {
      width: 40%;
      .not-found-img {
        width: 100%;
      }
    }
  }
}
</style>
