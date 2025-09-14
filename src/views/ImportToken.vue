<template>
  <div class="container">
    <div class="logo">
      <span>🔑</span>
    </div>
    
    <h1>RoleToken 提取工具</h1>
    <p class="description">上传BIN文件从服务器响应中提取RoleToken并生成WebSocket链接</p>
    
    <div class="api-info">
      <h3>API信息</h3>
      <div class="api-details">
        <p><strong>端点:</strong> https://xxz-xyzw.hortorgames.com/login/authuser?_seq=1</p>
        <p><strong>方法:</strong> POST</p>
        <p><strong>请求体:</strong> BIN文件内容</p>
        <p><strong>响应:</strong> 二进制数据 (包含RoleToken)</p>
      </div>
    </div>
    
    <div class="upload-area" 
         id="uploadArea"
         @click="handleUploadAreaClick"
         @dragenter.prevent="highlight"
         @dragover.prevent="highlight"
         @dragleave.prevent="unhighlight"
         @drop.prevent="handleDrop">
      <div class="upload-icon">📂</div>
      <p class="upload-text">点击或拖放BIN文件到此处</p>
      <p class="upload-subtext">仅支持 .bin 格式文件</p>
      <input type="file" 
             ref="fileInput" 
             class="file-input" 
             accept=".bin"
             @change="handleFileChange">
    </div>
    
    <div class="file-info" v-if="selectedFile">{{ fileInfoText }}</div>
    
    <div class="progress-container" v-if="showProgress">
      <div class="progress-bar">
        <div class="progress" :style="{ width: progressPercent + '%' }"></div>
      </div>
      <div class="progress-text">{{ progressPercent }}%</div>
    </div>
    
    <div class="status-message" 
         :class="statusType" 
         v-if="showStatusMessage">{{ statusMessage }}</div>
    
    <button class="btn" 
            @click="uploadFile"
            :disabled="!selectedFile || isUploading">上传BIN文件提取Token</button>
    
    <div class="result-container" v-if="showResult">
      <div class="result-title">提取到的 RoleToken:</div>
      <div class="token-display">{{ extractedToken }}</div>
      <button class="copy-btn" @click="copyToken">复制Token</button>
      
      <div class="token-info">
        <h3>Token使用说明</h3>
        <p>将此Token用于WebSocket连接认证：</p>
        <div class="usage-example">
ws://xxz-xyzw.hortorgames.com/agent?p={"roleToken":"你的Token","sessId":175742937218840,"connId":1757429372191,"isRestore":0}
        </div>
      </div>
      
      <div class="token-info" style="margin-top: 20px; background-color: #e8f5e9;">
        <h3>完整WebSocket链接</h3>
        <div class="wss-link-display">{{ wssLink }}</div>
        <button class="copy-btn" 
                @click="copyWssLink"
                style="background: #4caf50; margin-top: 10px;">复制完整WSS链接</button>
      </div>
    </div>
    
    <div class="debug-info" v-if="showDebugInfo">
      <h3>调试信息:</h3>
      <div class="response-details">{{ debugDetails }}</div>
    </div>
    
    <div class="response-info" v-if="showResponseInfo">
      <h3>原始响应数据 (十六进制格式):</h3>
      <div class="response-details">{{ responseDetails }}</div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ImportToken',
  data() {
    return {
      selectedFile: null,
      progressPercent: 0,
      showProgress: false,
      statusMessage: '',
      statusType: '',
      // showStatus: false,
      showStatusMessage: false,
      extractedToken: '',
      showResult: false,
      responseDetails: '',
      showResponseInfo: false,
      debugDetails: '',
      showDebugInfo: false,
      wssLink: '',
      isUploading: false
    };
  },
  computed: {
    fileInfoText() {
      if (!this.selectedFile) return '';
      return `已选择: ${this.selectedFile.name} (${this.formatFileSize(this.selectedFile.size)})`;
    }
  },
  methods: {
    handleUploadAreaClick(e) {
      if (e.target !== this.$refs.fileInput) {
        this.$refs.fileInput.click();
      }
    },
    highlight() {
      document.getElementById('uploadArea').classList.add('dragover');
    },
    unhighlight() {
      document.getElementById('uploadArea').classList.remove('dragover');
    },
    handleDrop(e) {
      this.unhighlight();
      const dt = e.dataTransfer;
      const files = dt.files;
      
      if (files.length) {
        this.handleFiles(files);
      }
    },
    handleFileChange() {
      if (this.$refs.fileInput.files.length) {
        this.handleFiles(this.$refs.fileInput.files);
      }
    },
    handleFiles(files) {
      const file = files[0];
      
      // 检查文件类型
      if (!file.name.toLowerCase().endsWith('.bin')) {
        this.showStatus('请选择.bin格式的文件', 'error');
        return;
      }
      
      this.selectedFile = file;
    },
    formatFileSize(bytes) {
      if (bytes === 0) return '0 Bytes';
      const k = 1024;
      const sizes = ['Bytes', 'KB', 'MB', 'GB'];
      const i = Math.floor(Math.log(bytes) / Math.log(k));
      return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
    },
    async uploadFile() {
      if (!this.selectedFile) {
        this.showStatus('请先选择文件', 'error');
        return;
      }
      
      // 重置状态
      this.isUploading = true;
      this.showProgress = true;
      this.progressPercent = 0;
      this.showStatus('正在上传文件...', '');
      
      try {
        // 读取文件内容
        const arrayBuffer = await this.readFileAsArrayBuffer(this.selectedFile);
        
        // 配置请求
        const xhr = new XMLHttpRequest();
        xhr.open('POST', 'https://xxz-xyzw.hortorgames.com/login/authuser?_seq=1', true);
        
        // 设置响应类型为arraybuffer以处理二进制数据
        xhr.responseType = 'arraybuffer';
        
        // 上传进度事件
        xhr.upload.addEventListener('progress', (e) => {
          if (e.lengthComputable) {
            const percentComplete = (e.loaded / e.total) * 100;
            this.progressPercent = Math.round(percentComplete);
          }
        });
        
        // 使用Promise包装XHR
        await new Promise((resolve, reject) => {
          // 请求完成事件
          xhr.addEventListener('load', () => {
            if (xhr.status >= 200 && xhr.status < 300) {
              try {
                // 处理二进制响应
                const arrayBuffer = xhr.response;
                if (arrayBuffer) {
                  // 提取RoleToken
                  this.extractRoleToken(arrayBuffer);
                  this.showStatus('Token提取成功！', 'success');
                  resolve();
                } else {
                  this.showStatus('上传成功，但响应为空', 'error');
                  reject(new Error('Empty response'));
                }
              } catch (e) {
                this.showStatus('处理响应时出错: ' + e.message, 'error');
                reject(e);
              }
            } else {
              this.showStatus('上传失败: ' + xhr.statusText, 'error');
              reject(new Error(`HTTP error: ${xhr.status}`));
            }
          });
          
          // 错误处理
          xhr.addEventListener('error', () => {
            this.showStatus('上传过程中发生错误', 'error');
            reject(new Error('Network error'));
          });
          
          // 设置请求头
          xhr.setRequestHeader('Content-Type', 'application/octet-stream');
          
          // 发送请求，将BIN文件内容作为请求体
          xhr.send(arrayBuffer);
        });
      } catch (error) {
        console.error('Upload error:', error);
      } finally {
        this.isUploading = false;
      }
    },
    readFileAsArrayBuffer(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = () => reject(reader.error);
        reader.readAsArrayBuffer(file);
      });
    },
    extractRoleToken(arrayBuffer) {
      try {
        // 将ArrayBuffer转换为Uint8Array以便处理
        const bytes = new Uint8Array(arrayBuffer);
        
        // 转换为ASCII字符串以便搜索
        let asciiString = '';
        for (let i = 0; i < bytes.length; i++) {
          // 只转换可打印的ASCII字符（32-126）
          if (bytes[i] >= 32 && bytes[i] <= 126) {
            asciiString += String.fromCharCode(bytes[i]);
          } else {
            asciiString += '.'; // 用点号表示不可打印字符
          }
        }
        
        // 显示调试信息
        this.debugDetails = 'ASCII预览:\n' + asciiString;
        this.showDebugInfo = true;
        
        // 搜索Token的位置 - 查找 "Token" 字符串
        const tokenIndex = asciiString.indexOf('Token');
        
        if (tokenIndex !== -1) {
          // 找到Token标记，提取Token值
          let tokenStart = tokenIndex + 5; // "Token"长度为5
          
          // 跳过可能的非Base64字符，直到找到Base64字符
          while (tokenStart < asciiString.length) {
            const char = asciiString[tokenStart];
            if (this.isBase64Char(char)) {
              break;
            }
            tokenStart++;
          }
          
          // 提取Base64 Token
          let tokenEnd = tokenStart;
          while (tokenEnd < asciiString.length && this.isBase64Char(asciiString[tokenEnd])) {
            tokenEnd++;
          }
          
          const tokenValue = asciiString.substring(tokenStart, tokenEnd);
          
          if (tokenValue.length > 0) {
            this.extractedToken = tokenValue;
            this.showResult = true;
            
            // 生成并显示完整的WSS链接
            this.generateAndDisplayWssLink(tokenValue);
            
            // 显示原始响应数据（十六进制格式）
            let formattedHex = '';
            for (let i = 0; i < bytes.length; i++) {
              if ((i + 1) % 16 === 0) formattedHex += '\n';
              formattedHex += bytes[i].toString(16).padStart(2, '0') + ' ';
            }
            this.responseDetails = formattedHex;
            this.showResponseInfo = true;
            
            // 平滑滚动到结果区域
            this.$nextTick(() => {
              document.querySelector('.result-container').scrollIntoView({ behavior: 'smooth' });
            });
          } else {
            this.showStatus('找到Token标记但未找到Token值', 'error');
          }
        } else {
          this.showStatus('在响应中未找到Token标记', 'error');
        }
      } catch (error) {
        this.showStatus('提取Token时发生错误: ' + error.message, 'error');
      }
    },
    isBase64Char(char) {
      // Base64字符集: A-Z, a-z, 0-9, +, /, =
      return /[A-Za-z0-9+/=]/.test(char);
    },
    showStatus(message, type) {
      this.statusMessage = message;
      this.statusType = type || '';
      // this.showStatus = true;
  this.showStatusMessage = true;
    },
    copyToken() {
      if (!this.extractedToken) return;
      
      navigator.clipboard.writeText(this.extractedToken)
        .then(() => this.showStatus('Token已复制到剪贴板', 'success'))
        .catch(() => this.showStatus('复制失败', 'error'));
    },
    copyWssLink() {
      if (!this.wssLink) return;
      
      navigator.clipboard.writeText(this.wssLink)
        .then(() => this.showStatus('WSS链接已复制到剪贴板', 'success'))
        .catch(() => this.showStatus('复制失败', 'error'));
    },
    generateAndDisplayWssLink(token) {
      // 生成随机的会话ID和连接ID
      const currentTime = Date.now();
      const sessId = currentTime * 100 + Math.floor(Math.random() * 100);
      const connId = currentTime + Math.floor(Math.random() * 10);
      
      // 构建WebSocket URL参数
      this.wssLink = `{"roleToken":"${token}","sessId":${sessId},"connId":${connId},"isRestore":0}`;
    }
  }
};
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

body {
  background: linear-gradient(135deg, #1a2980 0%, #26d0ce 100%);
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.container {
  background-color: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.25);
  width: 100%;
  max-width: 800px;
  padding: 30px;
  text-align: center;
  transition: all 0.3s ease;
}

.logo {
  width: 80px;
  height: 80px;
  background: linear-gradient(135deg, #1a2980 0%, #26d0ce 100%);
  border-radius: 50%;
  margin: 0 auto 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 36px;
  color: white;
}

h1 {
  color: #2c3e50;
  margin-bottom: 15px;
  font-weight: 700;
  font-size: 28px;
}

.description {
  color: #7f8c8d;
  margin-bottom: 25px;
  line-height: 1.6;
  font-size: 16px;
}

.upload-area {
  border: 3px dashed #ddd;
  border-radius: 12px;
  padding: 35px 25px;
  margin-bottom: 25px;
  cursor: pointer;
  transition: all 0.3s;
  background-color: #f8f9fa;
  position: relative;
}

.upload-area:hover, .upload-area.dragover {
  border-color: #3498db;
  background-color: rgba(52, 152, 219, 0.05);
}

.upload-icon {
  font-size: 60px;
  color: #3498db;
  margin-bottom: 15px;
}

.upload-text {
  font-size: 18px;
  color: #34495e;
  margin-bottom: 10px;
}

.upload-subtext {
  font-size: 14px;
  color: #7f8c8d;
}

.file-input {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  opacity: 0;
  cursor: pointer;
}

.file-info {
  margin-top: 15px;
  padding: 12px;
  background-color: #e8f4fc;
  border-radius: 8px;
  font-size: 14px;
  color: #2c3e50;
}

.btn {
  background: linear-gradient(135deg, #1a2980 0%, #26d0ce 100%);
  color: white;
  border: none;
  padding: 16px 25px;
  width: 100%;
  border-radius: 50px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0 5px 15px rgba(38, 208, 206, 0.4);
  margin-top: 10px;
}

.btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(38, 208, 206, 0.6);
}

.btn:active {
  transform: translateY(0);
}

.btn:disabled {
  background: #cccccc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.progress-container {
  margin: 20px 0;
}

.progress-bar {
  height: 10px;
  background-color: #e9ecef;
  border-radius: 5px;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: linear-gradient(135deg, #1a2980 0%, #26d0ce 100%);
  width: 0%;
  transition: width 0.4s ease;
}

.progress-text {
  text-align: right;
  font-size: 14px;
  color: #7f8c8d;
  margin-top: 8px;
}

.result-container {
  margin-top: 25px;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 12px;
  text-align: left;
}

.result-title {
  font-size: 16px;
  color: #2c3e50;
  margin-bottom: 12px;
  font-weight: 600;
}

.token-display {
  word-break: break-all;
  font-family: monospace;
  font-size: 14px;
  color: #27ae60;
  line-height: 1.5;
  padding: 12px;
  background-color: white;
  border-radius: 8px;
  border-left: 4px solid #27ae60;
  max-height: 200px;
  overflow-y: auto;
  white-space: pre-wrap;
}

.status-message {
  padding: 12px;
  border-radius: 8px;
  margin: 15px 0;
  text-align: center;
  font-weight: 500;
}

.success {
  background-color: rgba(39, 174, 96, 0.15);
  color: #27ae60;
}

.error {
  background-color: rgba(231, 76, 60, 0.15);
  color: #e74c3c;
}

.response-info {
  margin-top: 20px;
  padding: 15px;
  background-color: #f5f5f5;
  border-radius: 8px;
  font-size: 14px;
  color: #555;
  text-align: left;
}

.response-info h3 {
  margin-bottom: 10px;
  color: #333;
  font-size: 16px;
}

.response-details {
  font-family: monospace;
  white-space: pre-wrap;
  word-break: break-all;
}

.api-info {
  margin-top: 20px;
  padding: 15px;
  background-color: #f0f7ff;
  border-radius: 8px;
  text-align: left;
}

.api-info h3 {
  margin-bottom: 10px;
  color: #1a2980;
  font-size: 16px;
}

.api-details {
  font-size: 14px;
  color: #2c3e50;
}

.copy-btn {
  background: #27ae60;
  color: white;
  border: none;
  padding: 8px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.3s;
  margin-top: 10px;
}

.copy-btn:hover {
  background: #219653;
}

.token-info {
  margin-top: 15px;
  padding: 15px;
  background-color: #fff8e1;
  border-radius: 8px;
  border-left: 4px solid #ffc107;
}

.token-info h3 {
  color: #ff9800;
  margin-bottom: 10px;
  font-size: 16px;
}

.usage-example {
  font-family: monospace;
  background: #f5f5f5;
  padding: 10px;
  border-radius: 5px;
  margin-top: 10px;
  font-size: 13px;
  overflow-x: auto;
}

.debug-info {
  margin-top: 15px;
  padding: 15px;
  background-color: #e3f2fd;
  border-radius: 8px;
  border-left: 4px solid #2196f3;
}

.debug-info h3 {
  color: #1976d2;
  margin-bottom: 10px;
  font-size: 16px;
}

.wss-link-display {
  word-break: break-all;
  font-family: monospace;
  font-size: 14px;
  color: #2e7d32;
  line-height: 1.5;
  padding: 12px;
  background-color: white;
  border-radius: 8px;
  border-left: 4px solid #4caf50;
  margin-top: 10px;
}

@media (max-width: 576px) {
  .container {
    padding: 20px;
  }
  
  h1 {
    font-size: 24px;
  }
  
  .upload-area {
    padding: 25px 15px;
  }
}
</style>