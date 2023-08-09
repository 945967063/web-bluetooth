<template>
  <el-button type="primary" @click="connectBluetooth">连接蓝牙</el-button>
  <div v-if="isBluetooth">
    <el-input
      style="margin-top: 20px"
      v-model="textarea"
      :rows="3"
      type="textarea"
      placeholder="请输入给蓝牙设备发送的消息"
    />
    <el-button
      style="margin-top: 20px"
      type="primary"
      @click="sendMessage(textarea)"
      >发送消息</el-button
    >
    <div style="margin-top: 20px">接收到蓝牙返回的消息</div>

    <el-input
      v-model="characteristicValue"
      :rows="7"
      type="textarea"
      placeholder=""
    />
  </div>
</template>
<script setup lang="ts">
//蓝牙是否连接成功
const isBluetooth = ref(false);

const device = ref();

const bluetoothDevice = ref();

//发送数据
const unCharacteristic = ref();

//接收数据
const characteristicValue = ref("");

//消息
const textarea = ref('{"Command":16711937,"CommandName":"获取设备信息"}');
const connectBluetooth = async () => {
  (navigator as any).bluetooth.getAvailability().then((available: boolean) => {
    if (available) {
      console.log("设备支持蓝牙连接!", available);
    } else {
      ElNotification({
        message: "请检查蓝牙是否打开或设备不支持此功能",
        position: "bottom-right",
        type: "error",
      });
      console.log("设备不支持蓝牙");
    }
  });

  device.value = await (navigator as any).bluetooth.requestDevice({
    // filters: [
    //   // { name: 'SQRAY' }, //通过名称筛选
    //   // { services: ['0000ff02-0000-1000-8000-00805f9b34fb'] }, //通过UUID筛选，可以提供完整的蓝牙UUID或简短的16位或32位形式
    //   { namePrefix: 'JDY' }, //通过设备名称前缀过滤，如果通过名称前缀过滤的话则必须配置optionalServices注意参数内的大写字母都要改成小写
    // ],
    optionalServices: [0xffe0],
    acceptAllDevices: true, //设置acceptAllDevices代表查询所有蓝牙设备 也是必须配置optionalServices
  });
  //蓝牙连接成功
  isBluetooth.value = true;

  bluetoothDevice.value = device.value;
  device.value.addEventListener("gattserverdisconnected", onDisconnected); //监听设备断开连接
  let server = await bluetoothDevice.value.gatt.connect();
  let services = await server.getPrimaryServices();
  console.log(services, "services");
  const data = services.find((item: { uuid: string | string[] }) => {
    return item.uuid.includes("ffe0");
  });

  //如果如果不清楚服务UUID的话 可以先用getPrimaryServices获取服务的UUID 当然最简单的是去找公司的硬件工程师问一下 不过一般只做硬件工程师之前没有跟全栈这边配合过的话估计他们也不知道该给什么过来
  let service = await server.getPrimaryService(data.uuid);

  //获取设备所有信息特征值
  let characteristics = await service.getCharacteristics();
  //这个地方可以通过循环遍历获取你需要的特征值，根据properties来判断是读取还是写入
  console.log(characteristics, "characteristics");

  //读取特征值
  const charact = characteristics.find((item: { uuid: string | string[] }) => {
    return item.uuid.includes("ffe1");
  });

  //这里需要读的特征值UUID 还是可以提供完整的蓝牙UUID或简短的16位或32位形式  当然最简单的是去找公司的硬件工程师问一下 不过一般只做硬件工程师之前没有跟全栈这边配合过的话估计他们也不知道该给什么过来
  let characteristic = await service.getCharacteristic(charact.uuid);
  characteristic.startNotifications(); //开始监听

  //这里需要写入的特征值UUID 还是可以提供完整的蓝牙UUID或简短的16位或32位形式  当然最简单的是去找公司的硬件工程师问一下 不过一般只做硬件工程师之前没有跟全栈这边配合过的话估计他们也不知道该给什么过来
  unCharacteristic.value = await service.getCharacteristic(charact.uuid);

  //监听设备传过来的数据
  characteristic.addEventListener(
    "characteristicvaluechanged",
    async (e: any) => {
      //监听特征值变化
      let resBlob = new Blob([e.target.value]);
      let reader = new FileReader();
      reader.readAsText(resBlob, "utf-8");
      reader.onload = () => {
        console.log(reader.result, "reader.result");
        //蓝牙返回的数据有可能是一段一段的，就需要拼接起来

        //如果接受到多条消息，如何判断是哪一条呢
        //比如 你可以写个方法判断characteristicValue是否是一个json字符串，如果是的话就是你要的数据,并且清空characteristicValue，如果不是的话就继续拼接
        characteristicValue.value = characteristicValue.value + reader.result;
      };
    }
  );
};

//断开连接
function onDisconnected(event: any) {
  const device = event.target;
  isBluetooth.value = false;
  console.log(`设备: ${device} 已经断开连接`);
}

//发送消息
const sendMessage = async (value: string) => {
  console.log(value, "value");

  // 如果value.value字符串长度大于100，则分段发送，每次100个字符
  if (value.length > 100) {
    for (let i = 0; i < value.length; i += 100) {
      {
        let lines = value.slice(i, i + 100);
        let arrayBuffe = new TextEncoder().encode(lines + "\n");
        // console.log(unCharacteristic.value, lines, arrayBuffe);
        // 获取arrayBuffe有多少字节
        // console.log(arrayBuffe.byteLength);
        await unCharacteristic.value.writeValueWithoutResponse(arrayBuffe);
      }
    }
  } else {
    let lines = value.split("\n");
    let arrayBuffe = new TextEncoder().encode(lines + "\n");
    console.log(arrayBuffe, "arrayBuffe");

    // console.log(unCharacteristic.value, lines, arrayBuffe);
    // 获取arrayBuffe有多少字节
    // console.log(arrayBuffe.byteLength);
    await unCharacteristic.value.writeValueWithoutResponse(arrayBuffe);
  }
};
</script>
<style lang="scss" scoped></style>
