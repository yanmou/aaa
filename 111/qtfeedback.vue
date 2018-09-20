<template>
    <div class="qt-feedback">
        <!-- 问题反馈页面 -->
        <div class="qt-feedback-box">
            <div class="text-area">
                <textarea name="" id="qt-feedback-textarea" v-model="introduct" placeholder="请在此处输入你的问题或建议" maxlength="300">
                </textarea>
                <span class="letter-monitor">{{`${textareaVal} / 300`}}</span>
            </div>
            <!-- <div class="qt-upload-photos">
                <form class="upload-pic-form">
                    <div class="upload-pic-input-box">
                        <img :src="imgUrl" alt="" v-if="imgUrl">
                        <input type="file" class="upload-pic-input" @change="handleUpload($event)">
                    </div>
                    <div>
                        <p class="upload-title">图片上传</p>
                        <p class="upload-cont">请上传图片帮助我们快速处理你的问题</p>
                    </div>
                </form>
            </div> -->
            <div class="picture">
                <form class="picture">                
                    <ul class="picture-box clearfix">
                        <v-touch class="fl upload_img" v-for=" (img, idx) in pictures" :key="idx" @tap="prePicture(img,idx)" > 
                            <img :src="img" alt="" v-if="img">
                        </v-touch>
                    </ul>
                    <div class="clearfix add-picture">
                        <input type="file" class="upload-pic-input fl" @change="handleUpload($event)" accept="image/gif,image/jpeg,image/jpg,image/png,image/svg">
                        <div class="fl text" v-show ="!pictures.length">
                            <span>图片上传</span>
                            <p>请上传图片帮助我们快速处理你的问题</p>
                        </div>
                    </div>
                </form>
                <!-- <input class="file-input" id="image_uploads" type="file" multiple="multiple" accept="image/gif,image/jpeg,image/jpg,image/png,image/svg" /> -->
            </div>
        </div>

        <transition name="tip-transition" 
        enter-active-class="animated fadeIn"
        leave-active-class="animated fadeOut"
        :duration="150">
            <div v-if="preview.src" class="picture-preview">
                <v-touch class="content" @tap="closePre"><img :src="preview.src"></v-touch>
                <div class="preview-tool"><v-touch @tap="delPicture" class="del-btn"><img src="../../../assets/images/delete_icon.svg"></v-touch>
                </div>
            </div>
        </transition>
        <div>
            <span class="upload-botton" @click="uploadFeedback">提交</span>
        </div>
        <mt-popup v-model="popupVisible" class="popup-message-tip-box popup-message-feedback-box" popup-transition="popup-fade">
            <p class="title"> 感谢反馈</p>
            <p>您的反馈已经收到，我们将认真处理！</p>
        </mt-popup>
    </div>
</template>

<script>
import axios from 'axios';
import SparkMD5 from 'spark-md5';
import { appFeedback, getPolicyAndSign } from '@/assets/js/apolloGql.js';
import { getObjectURL } from '@/assets/js/util.js';
import '../../../styles/meui/meui.less';

export default {
    data() {
        return {
            introduct: '',
            textareaVal: 0,
            feedbackId: '', // 反馈id
            pictureUrl: '',
            sparkEnd: '',
            fileName: '',
            file: {},
            imgUrl: '',
            popupVisible: false,
            text: '',
            fontNum: 0,
            pictures: [],
            imgFiles: [],
            preview: {
                src: '',
                idx: 0
            }

        }
    },
    watch: {
        introduct(val) {
            this.textareaVal = val.length
        }
    },
    mounted() {
        // console.log( this.pictures.length )
        
    },
    methods: {
        prePicture(img, idx) {
            this.preview.src = img;
            this.preview.idx = idx;
        },
        delPicture() {
            this.pictures.splice(this.preview.idx, 1);
            this.imgFiles.splice(this.preview.idx, 1);
            this.closePre();
        },
        closePre() {
            this.preview.src = '';
            this.preview.idx = 0;
        },
        // 提交
        uploadFeedback() {
            if (!this.imgUrl) {
                this.$toast({
                    message: '请上传图片',
                    duration: 3000
                })
            } else if (this.introduct) {
                this.appFeedback();
            }
            // this.appFeedback();
        },
        // 图片上传获取md5
        handleUpload(event) {
            const fileReader = new FileReader();
            const blobSlice = File.prototype.mozSlice || File.prototype.webkitSlice || File.prototype.slice;
            const chunkSize = 10485800;
            const file = event.target.files[0];
            // console.log(file)
            const chunks = Math.ceil(file.size / chunkSize);
            // console.log(chunks)
            // read in chunks of 2MB
            const spark = new SparkMD5.ArrayBuffer();
            let currentChunk = 0;
            this.imgUrl = getObjectURL(file);
            if (this.pictures.length < 5) {
                this.pictures.push(this.imgUrl);
            } else {
                this.$toast({
                    message: '上传失败，最多上传5张图片!',
                    duration: 3000
                })
                return false
            }
            fileReader.onload = (e) => {
                window.console.log('read chunk nr', currentChunk + 1, 'of', chunks);
                spark.append(e.target.result); // append binary string
                currentChunk += 1;
                // if (2 < chunks) {
                //     loadNext();
                //     this.$toast({
                //         message: '上传失败，请上传不大于2M的图片!',
                //         duration: 3000
                //     })
                // } else {
                //     this.fileName = file.name;
                //     this.file = file;
                //     this.sparkEnd = spark.end();
                // }
                if (currentChunk < chunks) {
                    loadNext();
                    this.$toast({
                        message: '上传失败，请上传不大于10M的图片!',
                        duration: 3000
                    })
                } else {
                    this.fileName = file.name;
                    this.file = file;
                    this.sparkEnd = spark.end();
                }
            };
            fileReader.onerror = () => {
                console.warn('oops, something went wrong.');
            };
            function loadNext() {
                const start = currentChunk * chunkSize;
                const end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;

                fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
            }
            loadNext();
            
            
        },
        // 提交反馈内容
        async appFeedback() {
            // 获取feedbackId
            this.$apollo.mutate({
                mutation: appFeedback,
                variables: {
                    content: this.introduct,
                },
            }).then(res => {
                const data = res.data.appFeedback;
                if (data && data.code === 200) {
                    this.feedbackId = data.data;
                    this.$apollo.query({
                        query: getPolicyAndSign,
                        variables: {
                            fileName: this.fileName,
                            md5: this.sparkEnd,
                            feedbackId: this.feedbackId,
                        },
                        fetchPolicy: 'network-only',
                    }).then(res => {
                        const data = res.data.getPolicyAndSign;
                        if (data && data.code === 200) {
                            // 提交upyun
                            const params = new FormData();  // 创建form对象
                            params.append('file', this.file);
                            params.append('policy', data.data.policy); // 添加form表单中其他数据
                            params.append('authorization', data.data.signature);
                            const config = {
                                headers: { 'Content-Type': 'multipart/form-data' },
                            };
                            axios.post(data.data.url, params, config)
                                .then(response => {
                                    if (response.data.code === 200) {
                                        // this.$messagebox.confirm(`您的反馈已经收到，我们将认真处理！`, ' 感谢反馈', {
                                        //      showConfirmButton: false,
                                        //      showCancelButton: false,
                                        //      closeOnClickModal: true
                                        // });
                                        this.popupVisible = true;
                                        this.closePop();
                                    } else {
                                        this.$toast({
                                            message: '操作失败，请稍后再试!',
                                            duration: 3000
                                        })
                                    }
                                });
                        }
                    })
                }
            });
        },
        closePop() {
            setTimeout(() => {
                this.popupVisible = false;
                this.$router.go(-1);
            }, 3000);
        },
    }
    
}
</script>

<style scoped lang="stylus">
.qt-feedback {
    width: 100%;
    height: 100%;
    background: url('../../../assets/images/register/register_bg.png') no-repeat;
    background-size: cover;
    font-size: 14px;
    color: #fff;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
}

.text-area {
    position: relative;
    margin-bottom: 6px;
}

.qt-feedback-box {
    padding: 15px;
}

#qt-feedback-textarea {
    position: relative;
    width: 100%;
    height: 190px;
    background: rgba(255, 255, 255, 0.05);
    border: solid 0.5px rgba(255, 255, 255, 0.1);
    border-radius: 6px;
    padding: 10px;
    color: #fff;
    text-align: justify;
    resize: none;
}

.upload-pic-form {
    display: flex;
    justify-content: flex-start;
}

.upload-pic-input {
    width: 51px;
    height: 51px;
    background: transparent;
    text-indent: -9999px;
    color: transparent;
    vertical-align: middle;
    
    background: url('../../../assets/images/sidebar/model/Upload_bat.png') no-repeat;
}

.upload-title {
    font-size: 13px;
    line-height: 26px;
    text-align: left;
    color: #ffffff;
    margin-top: 3px;
}

.upload-pic-input-box {
    width: auto;
    height: 51px;
    background-size: cover;
    margin-right: 10px;
    position: relative;

    img {
        width: 89%;
        height: 89%;
        margin-top: 3px;
        border-radius: 2px;
    }
}

.upload-cont {
    font-size: 11px;
    text-align: left;
    color: rgba(255, 255, 255, 0.5);
}

.upload-botton {
    display: inline-block;
    width: 200px;
    height: 40px;
    line-height: 40px;
    font-size: 16px;
    color: #fff;
    border-radius: 6px;
    background-image: linear-gradient(to right, #00e2c9, #00a9e4);
    margin-bottom: 45px;
}

.letter-monitor {
    position: absolute;
    right: 10px;
    bottom: 0;
    font-size: 11px;
    line-height: 3.18;
    text-align: center;
    color: rgba(255, 255, 255, 0.5);
}

.picture {
    margin-top: 1px;
    height: auto;
    font-size: 0;
    padding-left: 0.1rem;
    text-align: left;
    overflow hidden;
}

.picture .picture-box, .picture .add-picture {
    display: inline-block;
    height: 100%;
}
.add-picture{
    height :51px;
    width :auto;
    margin-top :0.5rem;
}
.picture .picture-box .upload_img {
    width: 3.6rem;
    height: 3.6rem;
    margin-right: 0.5rem;
    text-align: center;
    cursor: pointer;
    img{
        width: 89%;
        height: 89%;
        margin-top: 3px;
        border-radius: 2px;
    }
}
.upload_img{
    margin-top :0.5rem;
}
.add-picture{
    margin-top :0.8rem;
}
.picture .picture-box .upload_img img {
    height: 100%;
}
.picture .add-picture > div:first-child {
    width: 3.6rem;
    height: 3.6rem;
    background: url('../../../assets/images/sidebar/model/Upload_bat.png') no-repeat;
    background-size: contain;
    margin-right: 0.8rem;
}

.picture .add-picture > div:last-child > span {
    display: block;
    color: #fff;
    font-size: 0.9rem;
    padding-top: 0.8rem;
}

.picture .add-picture > div:last-child p {
    font-size: 0.8rem;
    letter-spacing: 0rem;
    color: rgba(255, 255, 255, 0.5);
}

.picture .file-input {
    visibility: hidden;
}

.fl {
    float: left;
}

.fr {
    float: right;
}

.clearfix {
    @css { * }zoom: 1;
}

.clearfix:after {
    display: block;
    overflow: hidden;
    clear: both;
    height: 0;
    visibility: hidden;
    content: '.';
}
.text{
    margin-left : 10px;
}
.picture-preview{
    position: absolute;
    width: 100vw;
    height: 100vh;
    top: 0px;
    left: 0px;
    z-index: 999;
    background: #000000;
    background-image: -webkit-gradient(linear, left bottom, left top, from(#05070d), to(#27294a));
    background-image: linear-gradient(0deg, #05070d 0%, #27294a 100%);
}
.picture-preview .content{
    height: calc(100vh - 5rem);
}
.picture-preview .content img{
    position: absolute;
    width: 90vw;
    height: auto;
    top: 40%;
    left: 50%;
    transform: translate(-50%,-50%);
}
.picture-preview .preview-tool{
    position: absolute;
    width: 100vw;
    height: 5rem;
    bottom: 0px;
    text-align: center;

}
.picture-preview .preview-tool .del-btn{
    margin: 0 auto;
}
.picture-preview .preview-tool .del-btn img{
    width: 20px;
    height: auto;
}
</style>
