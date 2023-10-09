<template>
    <view class="container">
        <text class="title">欢迎来到主页！</text>
        <button @click="goToIndex">前往 Index 页面</button>
    </view>
</template>

<script>
export default {
    methods: {
        goToIndex() {
            uni.navigateTo({
                url: '/pages/index/index'
            });
        }
    }
}
</script>

<style scoped>
.container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
}
.title {
    font-size: 24px;
    font-weight: bold;
    margin-bottom: 20px;
}
</style>
