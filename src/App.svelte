<script>
  import { onMount } from "svelte";
  import { fly } from "svelte/transition";
  import Tailwind from "./Tailwind.svelte";
  import PDFPage from "./PDFPage.svelte";
  import Image from "./Image.svelte";
  import Text from "./Text.svelte";
  import Drawing from "./Drawing.svelte";
  import DrawingCanvas from "./DrawingCanvas.svelte";
  import prepareAssets, { fetchFont,Icons } from "./utils/prepareAssets.js";
  import {
    readAsArrayBuffer,
    readAsImage,
    readAsPDF,
    readAsDataURL
  } from "./utils/asyncReader.js";
  import axios from 'axios'
  import { ggID } from "./utils/helper.js";
  import { save } from "./utils/PDF.js";
  import createGuest from "cross-domain-storage/guest";
  import createHost  from "cross-domain-storage/host";
  const genID = ggID();
  let pdfFile;
  let pdfName = "";
  let iconName = "";
  let pages = [];
  let pagesScale = [];
  let allObjects = [];
  let currentFont = "Times-Roman";
  let focusId = null;
  let selectedPageIndex = -1;
  let saving = false;
  let addingDrawing = false;
  let formId;

  

var storageHost = createHost([
  {
    origin: "https://pdf-editor.smartnagar.org",
    allowedMethods: ["get", "set", "remove"],
  },
  {
    origin: "https://ebps.smartnagar.org",
    allowedMethods: ["get"],
  },
]);

  // for test purpose
  onMount(async () => {
    try {
      const urlParams = new URLSearchParams(window.location.search);
      formId = urlParams.get("form")
      const fetchPdf = await axios(`https://api.ebps.smartnagar.org/api/v1/form/${formId}/details`)
      const fetchPdfResponse = await fetchPdf.data
      const res = await fetch("data:application/pdf;base64," + fetchPdfResponse.base64)
 
      const pdfBlob = await res.blob();
      await addPDF(pdfBlob);
      selectedPageIndex = 0;
      setTimeout(() => {
        fetchFont(currentFont);
        prepareAssets();
      }, 5000);
    } catch (e) {
      console.log(e);
    }
  });
  async function onUploadPDF(e) {
    const files = e.target.files || (e.dataTransfer && e.dataTransfer.files);
    const file = files[0];
    if (!file || file.type !== "application/pdf") return;
    selectedPageIndex = -1;
    try {
      await addPDF(file);
      selectedPageIndex = 0;
    } catch (e) {
      console.log(e);
    }
  }
  async function addPDF(file) {
    try {
      const pdf = await readAsPDF(file);
      pdfName = file.name;
      pdfFile = file;
      const numPages = pdf.numPages;
      pages = Array(numPages)
        .fill()
        .map((_, i) => pdf.getPage(i + 1));
      allObjects = pages.map(() => []);
      pagesScale = Array(numPages).fill(1);
    } catch (e) {
      console.log("Failed to add pdf.");
      throw e;
    }
  }
  async function onUploadImage(e) {
    const file = e.target.files[0];
    if (file && selectedPageIndex >= 0) {
      addImage(file);
    }
    e.target.value = null;
  }

  function blobToFile(theBlob, fileName){
    //A Blob() is almost a File() - it's just missing the two properties below which we will add
    theBlob.lastModifiedDate = new Date();
    theBlob.name = fileName;
    return theBlob;
}

  async function onUploadIcon(e) {
      try {
        const res = await fetch(e.target.value);
      const pdfBlob = await res.blob();

var myFile =await  blobToFile(pdfBlob, "my-image.png");

      iconName = e.target.value
    if (myFile && selectedPageIndex >= 0) {
      addImage(myFile);
    }
    e.target.value = null;
    } catch(e) {
    e.target.value = null;
    }
  }
  
  async function addImage(file) {
    try {
      // get dataURL to prevent canvas from tainted
      const url = await readAsDataURL(file);
      const img = await readAsImage(url);
      const id = genID();
      const { width, height } = img;
      const object = {
        id,
        type: "image",
        width,
        height,
        x: 0,
        y: 0,
        payload: img,
        file
      };
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex ? [...objects, object] : objects
      );
    } catch (e) {
      console.log(`Fail to add image.`, e);
    }
  }
  function onAddTextField(xPostiton,y) {
    if (selectedPageIndex >= 0) {
      addTextField("New Text Field",xPostiton,y);
    }
  }
  function addTextField(text = "New Text Field",xPostiton,y) {
    const id = genID();
    fetchFont(currentFont);
    const object = {
      id,
      text,
      type: "text",
      size: 16,
      width: 0, // recalculate after editing
      lineHeight: 1.4,
      fontFamily: currentFont,
      x: 0,
      y: 0
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects
    );
  }

  function onAddDrawing() {
    if (selectedPageIndex >= 0) {
      addingDrawing = true;
    }
  }
  function addDrawing(originWidth, originHeight, path, scale = 1) {
    const id = genID();
    const object = {
      id,
      path,
      type: "drawing",
      x: 0,
      y: 0,
      originWidth,
      originHeight,
      width: originWidth * scale,
      scale
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects
    );
  }
  function selectFontFamily(event) {
    const name = event.detail.name;
    fetchFont(name);
    currentFont = name;
  }
  function selectPage(index) {
    selectedPageIndex = index;
    // onAddTextField();

  }
  function updateObject(objectId, payload) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.map(object =>
            object.id === objectId ? { ...object, ...payload } : object
          )
        : objects
    );
  }
  function deleteObject(objectId) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.filter(object => object.id !== objectId)
        : objects
    );
  }
  function onMeasure(scale, i) {
    pagesScale[i] = scale;
  }
  // FIXME: Should wait all objects finish their async work
  async function savePDF() {
    if (!pdfFile || saving || !pages.length) return;
    saving = true;
    try {
      await save(pdfFile, allObjects, pdfName, pagesScale,formId);
    } catch (e) {
      console.log(e);
    } finally {
      saving = false;
    }
  }


  function getClickPosition(e) {
    // debugger
    // var xPosition = e.clientX / 2 -80;
    // var yPosition = e.clientY- 80;
    // console.log(xPosition, yPosition);
    // onAddTextField(xPosition, yPosition)

}
</script>

<svelte:window
  on:dragenter|preventDefault
  on:dragover|preventDefault
  on:drop|preventDefault={onUploadPDF} />
<Tailwind />
<main class="flex flex-col items-center py-16 bg-gray-100 min-h-screen">
  <div
    class="fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center
    bg-gray-200 border-b border-gray-300">
    <input
      type="file"
      name="pdf"
      id="pdf"
      on:change={onUploadPDF}
      class="hidden" />
    <input
      type="file"
      id="image"
      name="image"
      class="hidden"
      on:change={onUploadImage} />
    <div
      class="relative mr-3 flex h-8 bg-gray-400 rounded-sm overflow-hidden
      md:mr-4">
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="image"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}>
        <img src="image.svg" alt="An icon for adding images" />
      </label>
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        for="text"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
        on:click={onAddTextField}>
        <img src="notes.svg" alt="An icon for adding text" />
      </label>
      <label
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500
        cursor-pointer"
        on:click={onAddDrawing}
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}>
        <img src="gesture.svg" alt="An icon for adding drawing" />
      </label>
    </div>
    <div class="justify-center my-3 mr-3 md:mr-3 w-200 max-w-xs hidden md:flex">
      <select
      name="iconName"
        placeholder="Select Icons"
        type="text"
        class="flex-grow bg-transparent"
       on:change={onUploadIcon} >
        <option value="null">Select Icons</option>
        <option value="/check.svg">Check</option>
        <option value="/radio.svg">Radio</option>
        </select>
    </div>
    <div class="justify-center mr-3 md:mr-4 w-full max-w-xs hidden md:flex">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName} />
    </div>
    <button
      on:click={savePDF}
      class="w-20 bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-3
      md:px-4 mr-3 md:mr-4 rounded"
      class:cursor-not-allowed={pages.length === 0 || saving || !pdfFile}
      class:bg-blue-700={pages.length === 0 || saving || !pdfFile}>
      {saving ? 'Saving' : 'Save'}
    </button>
    <a href="http://localhost:3000">
      <img
      style="width:2.1rem;height:2.1rem;"
        src="/logo.png"
        alt="Ebps" />
    </a>
  </div>
  {#if addingDrawing}
    <div
      transition:fly={{ y: -200, duration: 500 }}
      class="fixed z-10 top-0 left-0 right-0 border-b border-gray-300 bg-white
      shadow-lg"
      style="height: 50%;">
      <DrawingCanvas
        on:finish={e => {
          const { originWidth, originHeight, path } = e.detail;
          let scale = 1;
          if (originWidth > 500) {
            scale = 500 / originWidth;
          }
          addDrawing(originWidth, originHeight, path, scale);
          addingDrawing = false;
        }}
        on:cancel={() => (addingDrawing = false)} />
    </div>
  {/if}
  {#if pages.length}
    <div class="flex justify-center px-5 w-full md:hidden">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName} />
    </div>
    <div class="w-full">
      {#each pages as page, pIndex (page)}
        <div
          class="p-5 w-full flex flex-col items-center overflow-hidden"
          on:mousedown={() => selectPage(pIndex)}
          on:touchstart={() => selectPage(pIndex)}>
          <div
            class="relative shadow-lg"
            class:shadow-outline={pIndex === selectedPageIndex}>
            <PDFPage
              on:measure={e => onMeasure(e.detail.scale, pIndex)}
              {page} />
            <div
              class="absolute top-0 left-0 transform origin-top-left"
              style="transform: scale({pagesScale[pIndex]}); touch-action: none;">
              {#each allObjects[pIndex] as object (object.id)}
                {#if object.type === 'image'}
                  <Image
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    file={object.file}
                    payload={object.payload}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    height={object.height}
                    pageScale={pagesScale[pIndex]} />
                {:else if object.type === 'text'}
                  <Text
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    on:selectFont={selectFontFamily}
                    text={object.text}
                    x={object.x}
                    y={object.y}
                    size={object.size}
                    lineHeight={object.lineHeight}
                    fontFamily={object.fontFamily}
                    pageScale={pagesScale[pIndex]} />
                {:else if object.type === 'drawing'}
                  <Drawing
                    on:update={e => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    path={object.path}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    originWidth={object.originWidth}
                    originHeight={object.originHeight}
                    pageScale={pagesScale[pIndex]} />
                {/if}
              {/each}

            </div>
          </div>
        </div>
      {/each}
    </div>
  {:else}
    <div class="w-full flex-grow flex justify-center items-center">
      <span class=" font-bold text-3xl text-gray-500">Drag something here</span>
    </div>
  {/if}
</main>
