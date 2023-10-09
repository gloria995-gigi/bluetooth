<template>
    <view v-if="show" class="modal">
        <view class="modal-content">
            <text v-for="(device, index) in devices" :key="index" @tap="handleSelect(device)">
                {{ device[0].name }}
            </text>
        </view>
    </view>
</template>
  
<script>
export default {
    props: {
        show: Boolean, // 控制弹窗显示与隐藏的属性
        devices: Array // 设备信息数组
    },
    methods: {
        handleSelect(device) {
            this.$emit('select', device); // 发送选择事件
        }
    }
}
</script>
  
<style scoped>
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: #fff;
  padding: 20px;
  border-radius: 10px;
  text-align: center;
}
</style>
  
  