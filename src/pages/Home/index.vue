<template>
  <div class="p-home">
    <div class="p-home__left">
      <div class="p-home__tools">
        <button @click="addText">Text</button>
        <input
          v-model="editor.fontSize"
          type="text"
          placeholder="Font Size"
        >
        <input
          ref="input-file"
          type="file"
          multiple="false"
          @change="uploadPdf"
        >
        |
        <button
          @click="outputJson"
        >
          Output JSON
        </button>
      </div>
      <div class="p-home__canvas">
        <canvas
          id="pdf-editor-canvas"
        ></canvas>
        <canvas
          id="pdf-js"
          width="100"
          height="100"
        ></canvas>
      </div>
    </div>
    <div class="p-home__right">
      <div class="p-home__tools">
        offsetY
        <input
          v-model="output.offsetY"
          type="number"
        >
      </div>
      <div class="p-home__output">
        <textarea
          :value="output.json"
          :disabled="true"
        />
      </div>
    </div>
  </div>
</template>

<script>
import * as PdfJS from 'pdfjs-dist';
import { fabric } from 'fabric';

const fileToDataUrl = file => (
  new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (evt) => {
      resolve(evt.target.result);
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  })
);
const fileToDArray = file => (
  new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const fileDArray = new Uint8Array(reader.result);
      resolve(fileDArray);
    };
    reader.onerror = reject;
    reader.readAsArrayBuffer(file);
  })
);

const pageToImageDataUrl = (page) => (
  new Promise((resolve, reject) => {
    try {
      const scale = 1;
      const viewport = page.getViewport({ scale });

      // Prepare canvas using PDF page dimensions
      const canvas = document.getElementById('pdf-js');
      const context = canvas.getContext('2d');
      canvas.height = viewport.height;
      canvas.width = viewport.width;

      // Render PDF page into canvas context
      const task = page.render({
        canvasContext: context,
        viewport: viewport,
      });

      task.promise.then(() => {
        resolve({
          image: canvas.toDataURL('image/jpeg'),
          viewport,
        });
      });
    } catch (error) {
      reject(error);
    }
  })
);

export default {
  name: 'Home',
  data: () => ({
    canvas: null,
    editor: {
      fontSize: 11,
    },
    input: {
      image: null,
    },
    output: {
      offsetY: 3,
      json: '',
    },
  }),
  mounted() {
    // this.setupCanvas();
    this.setupPdfJs();
    this.setupKeyboardShortcut();
  },
  methods: {
    setupCanvas({ width = 500, height = 500 } = {}) {
      this.canvas = new fabric.Canvas('pdf-editor-canvas', {
        width,
        height,
      });
    },
    async setupPdfJs() {
      const pdfjsWorker = await import('pdfjs-dist/build/pdf.worker.entry');
      PdfJS.GlobalWorkerOptions.workerSrc = pdfjsWorker;
    },
    addText() {
      const sampleText = 'This is a text';
      const textObject = new fabric.Text(sampleText, {
        top: 10,
        left: 10,
        fontSize: this.editor.fontSize,
        fontFamily: 'Arial',
        hasControl: false,
        originY: 'bottom',
      });
      this.canvas.add(textObject);
    },
    async uploadPdf() {
      console.log(this.$refs['input-file'].files);
      const file = await fileToDArray(this.$refs['input-file'].files[0]);
      const pdfFile = await PdfJS.getDocument(file).promise;
      console.log(pdfFile);
      const firstPage = await pdfFile.getPage(1);
      const {
        image: imageFirstPage,
        viewport,
      } = await pageToImageDataUrl(firstPage);

      this.setupCanvas(viewport);

      fabric.Image.fromURL(imageFirstPage, (image) => {
        this.canvas.add(image);
      }, {
        // lockMovementX: true,
        // lockMovementY: true,
        selectable: false,
      });
    },
    async addImage() {
      console.log(this.$refs['input-file'].files);
      const file = await fileToDataUrl(this.$refs['input-file'].files[0]);
      fabric.Image.fromURL(file, (image) => {
        this.canvas.add(image);
      });
    },
    setupKeyboardShortcut() {
      document.onkeydown = (e) => {
        switch (e.key) {
          case 'ArrowUp':
            if(this.canvas.getActiveObject()){
              this.canvas.getActiveObject().top -= 5;
              this.canvas.renderAll();
            }
            break;
          case 'ArrowDown':
            if(this.canvas.getActiveObject()){
              this.canvas.getActiveObject().top += 5;
              this.canvas.renderAll(); 
            }
            break;
          case 'ArrowLeft':
            if(this.canvas.getActiveObject()){
              this.canvas.getActiveObject().left -= 5; 
              this.canvas.renderAll();
            }
            break;
          case 'ArrowRight':
            if(this.canvas.getActiveObject()){
              this.canvas.getActiveObject().left += 5; 
              this.canvas.renderAll();
            }
            break;
          case 'Backspace':
          case 'Delete':
            if(this.canvas.getActiveObject()){
              this.canvas.remove(this.canvas.getActiveObject()); 
            }
            break;
        }
      };
    },
    outputJson() {
      const { objects } = this.canvas.toJSON();
      const textResult = objects.filter(el => el.type === 'text');
      const { height: imageHeight } = objects.find(el => el.type === 'image');

      const result = textResult.map(el => ({
        x: el.left,
        y: imageHeight - el.top + this.output.offsetY,
      }));
      this.output.json = JSON.stringify(result, null, '  ');
    },
  },
};
</script>

<style>
#pdf-editor-canvas, #pdf-js {
  border: 1px solid black;
}
#pdf-js {
  display: none;
}
.p-home {
  display: flex;
}
.p-home__right, .p-home__left {
  width: 50%;
}
.p-home__output {
  display: flex;
  width: 100%;
  min-height: 100vh;
  white-space: pre-line;
  word-wrap: break-word;
}
.p-home__output textarea {
  width: 100%;
  height: calc(100vh - 100px);
}
</style>