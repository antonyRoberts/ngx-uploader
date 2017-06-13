# ngx-uploader

Angular 2+ File Uploader

http://ngx-uploader.com

**This module has been completely rewritten from scratch with the version `3.0.0`.**

If you are looking for documentation for version prior to `3.0.0`, please check [2.x.x](https://github.com/jkuri/ngx-uploader/tree/2.x.x) branch.

## Installation

1. Add `ngx-uploader` module as dependency to your project.

```
yarn add ngx-uploader
```

2. Include `NgUploaderModule` into your main AppModule or in module where you will use it.

```
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { NgUploaderModule } from 'ngx-uploader';

@NgModule({
  imports: [
    BrowserModule,
    NgUploaderModule
  ],
  declarations: [ AppComponent ],
  exports: [ AppComponent ]
})
export class AppModule {}
```

## Data Structures of Events and Uploaded Files

```ts
export interface UploadProgress {
  status: UploadStatus; // current status of upload for specific file (Queue | Uploading | Done | Canceled)
  data?: {
    percentage: number; // percentage of upload already completed
    speed: number; // current upload speed per second in bytes
    speedHuman: string; // current upload speed per second in human readable form
  };
}

export interface UploadFile {
  id: string; // unique id of uploaded file instance
  fileIndex: number; // fileIndex in internal ngx-uploader array of files
  lastModifiedDate: Date; // last modify date of the file (Date object)
  name: string; // original name of the file
  size: number; // size of the file in bytes
  type: string; // mime type of the file
  progress: UploadProgress;
  response?: any; // response when upload is done (parsed JSON or string)
}

// output events emitted by ngx-uploader
export interface UploadOutput {
  type: 'addedToQueue' | 'allAddedToQueue' | 'uploading' | 'done' | 'removed' | 'start' | 'cancelled' | 'dragOver' | 'dragOut' | 'drop';
  file?: UploadFile;
}

// input events that user can emit to ngx-uploader
export interface UploadInput {
  type: 'uploadAll' | 'uploadFile' | 'cancel' | 'cancelAll';
  url?: string; // URL to upload file to
  method?: string; // method (POST | PUT)
  id?: string; // unique id of uploaded file
  fieldName?: string; // field name (default 'file')
  fileIndex?: number; // fileIndex in internal ngx-uploader array of files
  file?: UploadFile; // uploading file
  data?: { [key: string]: string | Blob }; // custom data sent with the file
  headers?: { [key: string]: string }; // custom headers
  concurrency?: number; // concurrency of how many files can be uploaded in parallel (default is 0 which means unlimited)
  withCredentials?: boolean; // apply withCredentials option
}
```

## Example

**You can always run working example by cloning this repository, building project with `yarn build:prod` and running server with `node ./dist/server.js`.**

### Component Code

```ts
import { Component, EventEmitter } from '@angular/core';
import { UploadOutput, UploadInput, UploadFile, humanizeBytes } from 'ngx-uploader';

@Component({
  selector: 'app-home',
  templateUrl: 'app-home.component.html'
})
export class AppHomeComponent {
  formData: FormData;
  files: UploadFile[];
  uploadInput: EventEmitter<UploadInput>;
  humanizeBytes: Function;
  dragOver: boolean;

  constructor() {
    this.files = []; // local uploading files array
    this.uploadInput = new EventEmitter<UploadInput>(); // input events, we use this to emit data to ngx-uploader
    this.humanizeBytes = humanizeBytes;
  }

  onUploadOutput(output: UploadOutput): void {
    console.log(output); // lets output to see what's going on in the console

    if (output.type === 'allAddedToQueue') { // when all files added in queue
      // uncomment this if you want to auto upload files when added
      // const event: UploadInput = {
      //   type: 'uploadAll',
      //   url: '/upload',
      //   method: 'POST',
      //   data: { foo: 'bar' },
      //   concurrency: 0
      // };
      // this.uploadInput.emit(event);
    } else if (output.type === 'addedToQueue') {
      this.files.push(output.file); // add file to array when added
    } else if (output.type === 'uploading') {
      // update current data in files array for uploading file
      const index = this.files.findIndex(file => file.id === output.file.id);
      this.files[index] = output.file;
    } else if (output.type === 'removed') {
      // remove file from array when removed
      this.files = this.files.filter((file: UploadFile) => file !== output.file);
    } else if (output.type === 'dragOver') { // drag over event
      this.dragOver = true;
    } else if (output.type === 'dragOut') { // drag out event
      this.dragOver = false;
    } else if (output.type === 'drop') { // on drop event
      this.dragOver = false;
    }
  }

  startUpload(): void {  // manually start uploading
    const event: UploadInput = {
      type: 'uploadAll',
      url: '/upload',
      method: 'POST',
      data: { foo: 'bar' },
      concurrency: 1 // set sequential uploading of files with concurrency 1
    }

    this.uploadInput.emit(event);
  }

  cancelUpload(id: string): void {
    this.uploadInput.emit({ type: 'cancel', id: id });
  }
}
```

### Template Code

For whole template code please check [here](https://github.com/jkuri/ngx-uploader/tree/master/src/app/components/app-home/app-home.component.html).

```html
`<div class="drop-container" ngFileDrop (uploadOutput)="onUploadOutput($event)" [uploadInput]="uploadInput" [ngClass]="{ 'is-drop-over': dragOver }">
  <h1>Drag & Drop</h1>
</div>

<label class="upload-button">
  <input type="file" ngFileSelect (uploadOutput)="onUploadOutput($event)" [uploadInput]="uploadInput" multiple>
  or choose file(s)
</label>

<button type="button" class="start-upload-btn" (click)="startUpload()">
  Start Upload
</button>
```
### Advanced Example with plain JSON
#### Component
```
// advanced-demo.component.ts
import { Component, NgZone, Inject, EventEmitter } from '@angular/core';
import { NgUploaderOptions, UploadedFile, UploadRejected } from 'ngx-uploader';

@Component({
  selector: 'advanced-demo',
  templateUrl: 'advanced-demo.component.html'
})
export class AdvancedDemoComponent {
  options: NgUploaderOptions;
  response: any;
  sizeLimit: number = 1000000; // 1MB
  previewData: any;
  errorMessage: string;
  startUploadEvent: EventEmitter<string>;

  constructor(@Inject(NgZone) private zone: NgZone) {
    this.options = new NgUploaderOptions({
      url: 'http://api.ngx-uploader.com/upload',
      filterExtensions: true,
      allowedExtensions: ['txt', 'pdf'],
      maxSize: 2097152,
      data:{
        '@type': "File",
        "title": "My lorem.txt file",
        "file": {
          "data": "TG9yZW0gSXBzdW0u",
          "encoding": "base64",
          "filename": "lorem.txt",
          "content-type": "text/plain"
          }
        },
       customHeaders: {
         'Content-Type':'application/json',
         'Accept':'application/json'
      },
      autoUpload: false,
      plainJson: true,
      fieldName: 'file',
      fieldReset: true,
      maxUploads: 2,
      method: 'POST',
      previewUrl: true,
      withCredentials: false
    });

    this.startUploadEvent = new EventEmitter<string>();
  }

  startUpload() {
    this.startUploadEvent.emit("startUpload");
  }

  beforeUpload(ev : Event): void {
   
    let file: File =  ev.target['files'][0];
    let myReader: FileReader = new FileReader();

    myReader.onloadend = (e) => {
      let tmpB64String = myReader.result.split(',');
      this.options['data']['file']['data'] = tmpB64String[1] ;
      this.options['data']['file']['filename'] = file.name;
      this.options['data']['title'] = file.name;   
    }
    myReader.readAsDataURL(file);
  }

  handleUpload(data: any) {
    setTimeout(() => {
      this.zone.run(() => {
        this.response = data;
        if (data && data.response) {
          this.response = JSON.parse(data.response);
        }
      });
    });
  }

  handlePreviewData(data: any) {
    this.previewData = data;
  }
}
```
#### Template
```html
<!-- advanced-demo.component.html -->
<div class="button-container">
  <label class="upload-button is-pulled-left">
    <input type="file"
           class="hidden"
           ngFileSelect
           [options]="options"
           [events]="startUploadEvent"
           (onUpload)="handleUpload($event)"
           (onPreviewData)="handlePreviewData($event)"
           (beforeUpload)="beforeUpload($event)">
    Browse
  </label>
</div>

<p>
  Allowed extensions: <code><b>.jpg</b>, <b>.png</b></code>
</p>

<div>
  <button type="button" class="start-upload-button" (click)="startUpload()">Start Upload</button>
</div>

<div *ngIf="response">
  <pre><code>{{ response | json }}</code></pre>
</div>

<div *ngIf="errorMessage">
  <code>{{ errorMessage }}</code>
</div>

<div *ngIf="previewData && !response">
  <img [src]="previewData">
</div>
```
### LICENCE

MIT
