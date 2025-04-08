<template>
    <div class="camera">
        <h1>Heart Rate Measurement</h1>
        <h2>Videos</h2>
        <!-- <video ref="video" style="display: none;" autoplay></video> -->
        <video ref="video" autoplay></video>
        <h2>Frames</h2>
        <canvas ref="canvas" style="display: none;"></canvas>
        <div class="frames">
            <div class="graphs">
                <canvas ref="redCanvas"></canvas>
                <canvas ref="redGraphCanvas"></canvas>
            </div>
            <div class="graphs">
                <canvas ref="greenCanvas"></canvas>
                <canvas ref="greenGraphCanvas"></canvas>
            </div>
            <!-- <div class="graphs">
                <canvas ref="blueCanvas"></canvas>
                <canvas ref="blueGraphCanvas"></canvas>
            </div> -->
        </div>
        <canvas ref="combinedGraphCanvas"></canvas>

        <div class="controls">
            <button @click="startCamera">Start Camera</button>
            <button @click="stopCamera">Stop Camera</button>
            <button @click="toggleFlashlight">Toggle Flashlight</button>
        </div>
    </div>
</template>

<script>
import { useLog } from './LogHelper.vue';
import FFT from 'fft.js'; // 引入 FFT 库

const log = useLog();

export default {
    name: "Camera",
    data() {
        return {
            stream: null,   // 用于存储摄像头流
            frameInterval: null, // 用于存储帧提取的定时器
            graphXNum: 100,
            graphData: {
                red: [],
                green: [],
                blue: [],
                combined: [],
            },  // 用于存储波形数据
            videoFPS: null, // 用于存储帧率
            videoWidth: null, // 用于存储视频宽度
            videoHeight: null, // 用于存储视频高度
        };
    },
    methods: {
        async startCamera() {
            try {
                const constraints = {
                    video: {
                        facingMode: "environment", // 强制使用后置摄像头
                        width: { ideal: 640, max: 1920 }, // 尝试 1080p
                        height: { ideal: 480, max: 1080 },
                        frameRate: { ideal: 60, max: 120 }, // 请求更高的 FPS
                    },
                };

                this.stream = await navigator.mediaDevices.getUserMedia(constraints);
                this.$refs.video.srcObject = this.stream;

                const track = this.stream.getVideoTracks()[0];
                const settings = track.getSettings();
                this.videoFPS = settings.frameRate; // 获取帧率
                this.videoWidth = settings.width; // 获取视频宽度
                this.videoHeight = settings.height; // 获取视频高度
                log("Camera:", this.videoWidth, '*', this.videoHeight, '@', this.videoFPS, 'fps');

                // 开始提取帧
                this.startFrameExtraction();
            } catch (error) {
                log("Error accessing camera:", error);
            }
        },
        stopCamera() {
            if (this.stream) {
                this.stream.getTracks().forEach((track) => track.stop());
                this.stream = null;
            }
            clearInterval(this.frameInterval); // 停止帧提取
            this.frameInterval = null;
        },
        async toggleFlashlight() {
            if (!this.stream) {
                log("Camera is not started. Please start the camera first.");
                return;
            }

            const track = this.stream.getVideoTracks()[0];
            const capabilities = track.getCapabilities();

            if (!capabilities.torch) {
                log("Torch (flashlight) is not supported on this device.");
                return;
            }

            try {
                this.isFlashlightOn = !this.isFlashlightOn;
                await track.applyConstraints({
                    advanced: [{ torch: this.isFlashlightOn }],
                });
                log(`Flashlight turned ${this.isFlashlightOn ? "on" : "off"}`);
            } catch (error) {
                log("Error toggling flashlight:", error);
            }
        },
        startFrameExtraction() {
            // 清除之前的帧提取定时器
            if (this.frameInterval) {
                clearInterval(this.frameInterval);
            }

            // 获取 DOM 元素
            const video = this.$refs.video;
            const canvas = this.$refs.canvas;
            const redCanvas = this.$refs.redCanvas;
            const greenCanvas = this.$refs.greenCanvas;
            // const blueCanvas = this.$refs.blueCanvas;
            const redGraphCanvas = this.$refs.redGraphCanvas;
            const greenGraphCanvas = this.$refs.greenGraphCanvas;
            // const blueGraphCanvas = this.$refs.blueGraphCanvas;
            const combinedGraphCanvas = this.$refs.combinedGraphCanvas;

            // 获取上下文对象
            const ctx = canvas.getContext("2d");
            const redCtx = redCanvas.getContext("2d");
            const greenCtx = greenCanvas.getContext("2d");
            // const blueCtx = blueCanvas.getContext("2d");
            const redGraphCtx = redGraphCanvas.getContext("2d");
            const greenGraphCtx = greenGraphCanvas.getContext("2d");
            // const blueGraphCtx = blueGraphCanvas.getContext("2d");
            const combinedGraphCtx = combinedGraphCanvas.getContext("2d");

            // 设置 graphCanvas 尺寸
            redGraphCanvas.width = 800;
            redGraphCanvas.height = 200;
            greenGraphCanvas.width = 800;
            greenGraphCanvas.height = 200;
            // blueGraphCanvas.width = 800;
            // blueGraphCanvas.height = 200;
            combinedGraphCanvas.width = 800;
            combinedGraphCanvas.height = 200;

            video.addEventListener("loadeddata", () => {
                // 设置 canvas 尺寸
                canvas.width = video.videoWidth;
                canvas.height = video.videoHeight;

                // 设置 ROI 区域
                const roiWidthPercent = 0.5; // ROI 宽度为图像宽度的 50%
                const roiHeightPercent = 0.5; // ROI 高度为图像高度的 50%
                const roiWidth = Math.floor(canvas.width * roiWidthPercent); // ROI 的宽度
                const roiHeight = Math.floor(canvas.height * roiHeightPercent); // ROI 的高度
                const roiX = Math.floor((canvas.width - roiWidth) / 2); // ROI 的 X 坐标
                const roiY = Math.floor((canvas.height - roiHeight) / 2); // ROI 的 Y 坐标

                // 设置 canvas 的宽高为 ROI 的大小
                redCanvas.width = roiWidth;
                redCanvas.height = roiHeight;
                greenCanvas.width = roiWidth;
                greenCanvas.height = roiHeight;
                // blueCanvas.width = roiWidth;
                // blueCanvas.height = roiHeight;

                redGraphCanvas.height = roiHeight;
                greenGraphCanvas.height = roiHeight;
                // blueGraphCanvas.height = roiHeight;
                combinedGraphCanvas.height = roiHeight;

                this.frameInterval = setInterval(() => {
                    if (!video.paused && !video.ended) {
                        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);    // 绘制视频帧到 canvas 上
                        const frame = ctx.getImageData(roiX, roiY, roiWidth, roiHeight); // 仅提取 ROI 区域

                        const redFrame = ctx.createImageData(roiWidth, roiHeight);  // 创建新的图像数据对象
                        const greenFrame = ctx.createImageData(roiWidth, roiHeight);
                        // const blueFrame = ctx.createImageData(roiWidth, roiHeight);

                        let redData = [];
                        let redChannelSum = 0;
                        let greenData = [];
                        let greenChannelSum = 0;
                        // let blueData = [];
                        // let blueChannelSum = 0;
                        let combinedData = [];
                        let combinedChannelSum = 0;

                        for (let i = 0; i < frame.data.length; i += 4) {
                            // 红色通道
                            redFrame.data[i] = frame.data[i];
                            redFrame.data[i + 1] = 0;
                            redFrame.data[i + 2] = 0;
                            redFrame.data[i + 3] = 255;
                            redData.push(frame.data[i]);
                            redChannelSum += frame.data[i];

                            // 绿色通道
                            greenFrame.data[i] = 0;
                            greenFrame.data[i + 1] = frame.data[i + 1];
                            greenFrame.data[i + 2] = 0;
                            greenFrame.data[i + 3] = 255;
                            greenData.push(frame.data[i + 1]);
                            greenChannelSum += frame.data[i + 1];

                            // 蓝色通道
                            // blueFrame.data[i] = 0;
                            // blueFrame.data[i + 1] = 0;
                            // blueFrame.data[i + 2] = frame.data[i + 2];
                            // blueFrame.data[i + 3] = 255;
                            // blueData.push(frame.data[i + 2]);
                            // blueChannelSum += frame.data[i + 2];

                            // 组合通道
                            let redWeight = 0.5;
                            let greenWeight = 1 - redWeight;
                            const combinedValue = redWeight * frame.data[i] + greenWeight * frame.data[i + 1];
                            combinedData.push(combinedValue);
                            combinedChannelSum += combinedValue;
                        }

                        // 计算平均值
                        const redChannelAverage = redChannelSum / (redData.length || 1);
                        const greenChannelAverage = greenChannelSum / (greenData.length || 1);
                        // const blueChannelAverage = blueChannelSum / (blueData.length || 1);
                        const combinedChannelAverage = combinedChannelSum / (combinedData.length || 1);

                        // 绘制图像
                        redCtx.putImageData(redFrame, 0, 0);
                        greenCtx.putImageData(greenFrame, 0, 0);
                        // blueCtx.putImageData(blueFrame, 0, 0);

                        // 更新图形数据
                        this.graphData.red.push(redChannelAverage);
                        this.graphData.green.push(greenChannelAverage);
                        // this.graphData.blue.push(blueChannelAverage);
                        this.graphData.combined.push(combinedChannelAverage);

                        // 限制数据长度，保持图形流畅
                        if (this.graphData.red.length > this.graphXNum) {
                            this.graphData.red.shift();
                            this.graphData.green.shift();
                            // this.graphData.blue.shift();
                            this.graphData.combined.shift();
                        }

                        // 绘制图形
                        this.heartRate(redGraphCtx, this.graphData.red, "red");
                        this.heartRate(greenGraphCtx, this.graphData.green, "green");
                        // this.heartRate(blueGraphCtx, this.graphData.blue, "blue");
                        this.heartRate(combinedGraphCtx, this.graphData.combined, "purple"); // 绘制加权通道
                    }
                }, 1000/this.videoFPS); // 根据帧率设置间隔
            });
        },

        heartRate(ctx, data, color) {
            ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);   // 清除画布
            const smoothedData = this.smoothData(data);
 
            this.drawGraph(ctx, data, color); // 绘制波形图
            this.drawGraph(ctx, smoothedData, "blue"); // 绘制平滑波形图
            this.showHeartRate(ctx, smoothedData, "black"); // 显示心率
        },

        drawGraph(ctx, data, color) {
            // ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);   // 清除画布

            ctx.strokeStyle = color;
            ctx.beginPath();
            data.forEach((value, index) => {
                const x = (index / this.graphXNum) * ctx.canvas.width;
                const y = ctx.canvas.height - (value / 255) * ctx.canvas.height;
                if (index === 0) {
                    ctx.moveTo(x, y);
                } else {
                    ctx.lineTo(x, y);
                }
            });
            ctx.stroke();
        },

        calculateHeartRate(data) {
            // 计算峰峰值
            const maxValue = Math.max(...data);
            const minValue = Math.min(...data);
            const peakToPeak = maxValue - minValue;

            // 计算周期
            let period = 0;
            let heartRate = 0;

            if (data.length > 1) {
                // 改进的峰值检测逻辑
                let peaks = [];
                const threshold = (maxValue - minValue) * 0.5; // 设置一个阈值
                for (let i = 1; i < data.length - 1; i++) {
                    if (
                        data[i] > data[i - 1] &&
                        data[i] > data[i + 1] &&
                        data[i] > minValue + threshold
                    ) {
                        peaks.push(i);
                    }
                }

                if (peaks.length > 1) {
                    const averagePeakDistance = (peaks[peaks.length - 1] - peaks[0]) / (peaks.length - 1);
                    period = averagePeakDistance / this.videoFPS; // 周期 = 峰值间隔 / 采样率
                    heartRate = period > 0 ? 60 / period : 0; // 心率 = 60 / 周期
                }
            }

            // 返回心率和峰峰值
            return {
                heartRate,
                period,
                peakToPeak,
            };
        },

        showHeartRate(ctx, data, color) {
            // ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);   // 清除画布

            // 计算心率
            const { heartRate, period, peakToPeak } = this.calculateHeartRate(data);
            // const bpm = this.calculateFFT(data, this.videoFPS);
            const bpm = 0;

            // 显示心率和峰峰值
            ctx.fillStyle = color;
            ctx.font = "14px Arial";
            ctx.fillText(`Peak-to-Peak: ${peakToPeak.toFixed(2)}`, 10, 20);
            ctx.fillText(`Period: ${period.toFixed(2)} s`, 10, 40);
            ctx.fillText(`Heart Rate: ${heartRate.toFixed(2)} bpm`, 10, 60);
            if (bpm !== null) {
                ctx.fillText(`FFT: ${bpm.toFixed(2)} bpm`, 10, 80);
            }
        },

        // 平滑数据函数
        smoothData(data, windowSize = 5) {
            const smoothed = [];
            for (let i = 0; i < data.length; i++) {
                const start = Math.max(0, i - Math.floor(windowSize / 2));
                const end = Math.min(data.length, i + Math.floor(windowSize / 2) + 1);
                const window = data.slice(start, end);
                smoothed.push(window.reduce((sum, val) => sum + val, 0) / window.length);
            }
            return smoothed;
        },
        
        calculateFFT(data, sampleRate) {
            const fftSize = 64;
            const fft = new FFT(fftSize);

            // 确保输入数据长度固定
            const input = new Float32Array(fftSize).fill(0);
            const length = Math.min(data.length, fftSize);
            input.set(data.slice(0, length));

            // 去直流
            const mean = input.reduce((sum, val) => sum + val, 0) / fftSize;
            for (let i = 0; i < fftSize; i++) {
                input[i] -= mean;
            }

            const output = new Float32Array(fftSize * 2);
            fft.realTransform(output, input);
            fft.completeSpectrum(output);

            // 计算幅值
            const magnitudes = [];
            for (let i = 0; i < fftSize / 2; i++) {
                const re = output[2 * i];
                const im = output[2 * i + 1];
                magnitudes.push(Math.sqrt(re * re + im * im));
            }

            const frequencyResolution = sampleRate / fftSize;

            // 仅搜索心率可能范围
            const minBPM = 40;
            const maxBPM = 180;
            const minIndex = Math.floor((minBPM / 60) / frequencyResolution);
            const maxIndex = Math.floor((maxBPM / 60) / frequencyResolution); // 使用 Math.floor 来避免越界

            let peakIndex = -1;
            let maxAmplitude = 0;
            for (let i = minIndex; i <= maxIndex; i++) {
                if (magnitudes[i] > maxAmplitude) {
                    maxAmplitude = magnitudes[i];
                    peakIndex = i;
                }
            }

            // 如果找到了有效的峰值
            if (peakIndex !== -1) {
                const peakFreq = peakIndex * frequencyResolution;
                const bpm = peakFreq * 60;
                return bpm;
            }

            // 如果没有找到有效的心率峰值，返回 null
            return null;
        },
    },
};
</script>

<style scoped>
.camera {
    text-align: center;
}
video {
    width: auto;
    height: auto;
    /* max-width: 600px; */
    margin: 20px 0;
    border: 1px solid #ccc;
}
canvas {
    width: auto;
    /* max-width: 600px; */
    height: auto;
    margin: 0px 0;
    border: 1px solid #ccc;
    box-sizing: border-box; /* 包括内边距和边框 */
}
.frames {
    display: flex;
    flex-direction: column;
    justify-content: space-around;
    align-items: center;
    flex-wrap: wrap;
    gap: 5px;
}
/* .frames canvas {
    flex: 1;
    width: auto;
    max-width: none;
    height: auto;
    border: 1px solid #ccc;
    box-sizing: border-box;
} */
.graphs {
    display: flex;
    flex-direction: row;
    justify-content: space-around;
    align-items: center;
    gap: 5px;
}
.graphs canvas {
    width: auto;
    /* max-width: none; 限制每个元素的最大宽度 */
    height: auto; /* 保持宽高比 */
    border: 1px solid #ccc; /* 添加边框 */
    box-sizing: border-box; /* 包括内边距和边框 */
}
.controls {
    margin: 20px 0;
}
button {
    margin: 5px;
    padding: 10px 20px;
    font-size: 16px;
    cursor: pointer;
}
</style>