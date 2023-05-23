<script setup>
  import { ref, onMounted } from 'vue'
  import pic01 from '../../public/01.jpeg'

  const picture = ref(new Image())

  const canvas = ref(null)
  
  const cxt = ref(null)
  // 多少份
  const count = ref(3)
  // 以高度
  const clipHeight = ref(100)
  // 图片质量
  const quality = ref('1')
  const imgType = ref('image/jpeg')

  // 1 以份数 2 以高度
  const way = ref('1')

  // 剪切后生成的canvas
  const resultCanvasBox = ref(null)

  // 剪切后生成的图片
  const resultImgBox = ref(null)
  const imgFileArr = ref([])

  picture.value.onload = () => {
    canvas.value.height = picture.value.height
    canvas.value.width = picture.value.width
    cxt.value.drawImage(picture.value, 0, 0, canvas.value.width, canvas.value.height)
  }


  function fileChange (e) {
    const files = e.target.files
    picture.value.src = window.URL.createObjectURL(files[0])
  }

  function canvasToBlob () {
    const resultList = document.querySelectorAll('.result-canvas')
    const promiseArr = []

    for (let i = 0; i < resultList.length; i++) {
      promiseArr.push(new Promise((resolve) => {
        // canvas转换成base64
        // let base64 = canvas2.toDataURL("image/jpeg", 1);
        resultList[i].toBlob(
          (blob) => {
            blob.name = 'test'
            imgFileArr.value.push(blob)
            const data = URL.createObjectURL(blob)
            // 生成一个临时地址
            resultImgBox.value.getElementsByTagName('img')[i].src = data
            resolve()
          },
          imgType.value,
          Number.parseFloat(quality.value)
        )
      }))
    }

    Promise.all(promiseArr).then(() => {
      console.log('裁剪完成')
      console.log(imgFileArr.value)
    })
  }

  function onClip () {
    const resultCanvasDocFragment = document.createDocumentFragment()
    const resultImgDocFragment = document.createDocumentFragment()

    let height = 0
    // 以份数
    if (way.value === '1') {
      count.value = isNaN(Number.parseInt(count.value))
      ? 1
      : count.value
      height = Number.parseInt((canvas.value.height - (canvas.value.height % count.value)) / count.value)
    }
    
    // 以高度
    if (way.value === '2') {
      count.value = (canvas.value.height % clipHeight.value > 0)
        ? Number.parseInt(canvas.value.height / clipHeight.value) + 1
        : Number.parseInt(canvas.value.height / clipHeight.value)

      height = clipHeight.value
    }

    resultCanvasBox.value.innerHTML = ''
    resultImgBox.value.innerHTML = ''

    console.log(count.value, height)

    for (let i = 0; i < count.value; i++) {
      let newCanvas = document.createElement('canvas')
      newCanvas.className = 'result-canvas'

      let sy = i * height;
      // 判断是否最后一个
      if (i === count.value - 1) {
        // 为份数
        if (way.value === '1') {
          height += (canvas.value.height % count.value)
        }
        // 为高度
        if (way.value === '2') {
          height = (canvas.value.height % clipHeight.value === 0)
            ? height
            : canvas.value.height % clipHeight.value
        }
      }

      newCanvas.width = canvas.value.width
      newCanvas.height = height
      
      let newCxt = newCanvas.getContext('2d')
      // 返回一个ImageData对象，用来描述canvas区域隐含的像素数据
      let dataImg = cxt.value.getImageData(0, sy, newCanvas.width, height)
      // 然后通过 putImageData 将图像数据放回画布
      newCxt.putImageData(dataImg, 0, 0, 0, 0, newCanvas.width, height)

      resultCanvasDocFragment.appendChild(newCanvas)
      resultImgDocFragment.appendChild(new Image())
    }

    resultCanvasBox.value.appendChild(resultCanvasDocFragment)
    resultImgBox.value.appendChild(resultImgDocFragment)

    canvasToBlob()
  }

  function downloadFile ({ data, name, type = 'application/jpeg', suffix = 'jpeg' } = {}) {
    const blob = new Blob([data], { type })
    const fileName = `${name}.${suffix}`
    if ('download' in document.createElement('a')) { // 非IE下载
      const elink = document.createElement('a')
      elink.download = fileName
      elink.style.display = 'none'
      elink.href = URL.createObjectURL(blob)
      document.body.appendChild(elink)
      elink.click()
      URL.revokeObjectURL(elink.href) // 释放URL 对象
      document.body.removeChild(elink)
    } else { // IE10+下载
      navigator.msSaveBlob(blob, fileName)
    }
  }

  function download () {
    imgFileArr.value.forEach((item, index) => {
      downloadFile({
        data: item,
        name: '0' + index + 1,
      })
    })
  }

  onMounted(() => {
    canvas.value = document.getElementById('canvas-original')
    cxt.value = canvas.value.getContext('2d')

    resultCanvasBox.value = document.querySelector('.canvas-box')
    resultImgBox.value = document.querySelector('.img-box')

    picture.value.src = pic01
  })
</script>

<template>
  <main class="main">
    <div>
			<input @change="fileChange($event)" type="file" multiple="multiple">
		</div>
		<div class="">
			<div>
				<label>哪种方式：</label>
				<select v-model="way">
					<option value="1">以份数</option>
					<option value="2">以高度</option>
				</select>
			</div>
			
			<div v-if="way === '1'">
				<label>以份数：</label>
				<input v-model.number="count" type="text">
			</div>

			<div v-if="way === '2'">
				<label>以高度：</label>
				<input v-model.number="clipHeight" type="text">
				<span>{{  }}</span>
			</div>

			<div class="">
				<label>图片质量0~1：</label>
				<input v-model.trim="quality" type="text">

				<label>图片格式：</label>
				<select v-model="imgType">
					<option value="image/jpeg">image/jpeg</option>
					<!-- <option value="image/jpeg">image/webp</option> -->
					<option value="image/png">image/png</option>
				</select>
				<span>--格式为png质量无效</span>
				<!-- 当请求图片格式为image/jpeg或者image/webp时用来指定图片展示质量 -->
			</div>

			<button @click="onClip">开始裁剪</button>
			<button @click="download">下载</button>
		</div>

		<div>
      <div>
        <span>height: {{ canvas ? canvas.height : 0 }}</span>
        <span>-</span>
        <span>width: {{ canvas ? canvas.width : 0 }}</span>
      </div>
			<div>
				<canvas id="canvas-original"></canvas>
			</div>

			<div class="img-box">

      </div>
      
      <div style="height: 2px; margin: 5px 0; background-color: aquamarine;"></div>

			<div class="canvas-box">

      </div>
		</div>
  </main>
</template>

<style scoped>
  .canvas-box{
    border: 1px solid rgb(134, 65, 65);
    min-height: 60px;
  }
  .canvas-box canvas{
    margin: 0 0 5px 0;
    box-shadow: 0 0 3px rgba(0, 0, 0, 0.2);
    font-size: 0;
    display: block;
  }
  .img-box {

    border: 1px solid gray;
    min-height: 60px;
  }
  .img-box img{
    display: block;
  }
  label{
    display: inline-block;
    width: 130px;
    text-align: right;
  }
</style>
