<script>
  import { onMount, afterUpdate } from "svelte";
  import * as gifList from "./giflist";
  import Button from "./Button.svelte";

  let frameCount = 0;
  export let croppedImage;
  export let step;
  const goBackStep = () => {
    step = 1;
  };
  let faceImage;
  export let imageSelection = "partyParrot";
  let imageFrameCount = 36;
  let imagePlayRate = 25;
  let loadedSelection = "partyParrot";
  let canvas, ctx;
  let positionsArr = [];
  let arr;

  let canvasOffset,
    offsetX,
    offsetY,
    canMouseX,
    canMouseY,
    isDragging = false,
    position = { x: 0, y: 0 },
    imageOrigPos = { ...position },
    clickOrigPos,
    playInterval;

  const onScroll = (e) => setOffsets();

  onMount(() => {
    canvas = document.getElementById("gifCanvas");
    ctx = canvas.getContext("2d");
    setOffsets();
    document.addEventListener("scroll", setOffsets);
    window.addEventListener("resize", setOffsets);
    extractGif().then((imgArr) => {
      arr = imgArr;
      positionsArr = [...gifList[imageSelection].positions];
      imagePlayRate = gifList[imageSelection].playSpeed;
      setFrame(0);
    });
  });

  afterUpdate(() => {
    console.log(imageSelection);
    if (faceImage !== croppedImage || imageSelection !== loadedSelection) {
      console.log(imageSelection, loadedSelection);
      faceImage = croppedImage;
      extractGif(imageSelection).then((imgArr) => {
        console.log(imgArr);
        loadedSelection = imageSelection;
        positionsArr = [...gifList[imageSelection].positions];
        imagePlayRate = gifList[imageSelection].playSpeed;
        arr = imgArr;
        setFrame(0);
      });
    }
  });

  function setOffsets() {
    canvasOffset = canvas.getBoundingClientRect();
    offsetX = canvasOffset.left;
    offsetY = canvasOffset.top;
  }

  function extractGif(imageSelection = loadedSelection) {
    const Img = [];
    let imgsLoaded = 0;
    const totalFrames = gifList[imageSelection].positions.length;
    console.log(imageSelection, totalFrames);
    imageFrameCount = totalFrames;
    return new Promise((resolve, reject) => {
      for (var i = 0; i < totalFrames; i++) {
        Img[i] = new Image();
        Img[i].src = `${imageSelection}/${i}.gif`;
        Img[i].onload = () => {
          imgsLoaded++;
          if (imgsLoaded === totalFrames) resolve(Img);
        };
      }
    });
  }

  function setFrame(frame) {
    ctx.clearRect(0, 0, 1000, 1000);
    console.log(positionsArr);
    frameCount = frame;
    console.log(frameCount, arr.length);
    if (frameCount > arr.length - 1) frameCount = 0;
    if (frameCount < 0) frameCount = arr.length - 1;
    if (arr.length == imageFrameCount) drawFrame(frameCount);
  }

  function drawAnimatedImage(frame, overrideCtx) {
    const targetCtx = overrideCtx ?? ctx;
    if (!!arr[frame]) {
      console.log("drawing parrot", arr[frame]);
      targetCtx.drawImage(arr[frame], 0, 0);
    }
    targetCtx.restore();
    return frame;
  }

  function drawOverlayImage(frameCount, overrideCtx) {
    if (faceImage) {
      const targetCtx = overrideCtx ?? ctx;
      const posX = positionsArr[frameCount]?.x ? positionsArr[frameCount].x : 0;
      const posY = positionsArr[frameCount]?.y ? positionsArr[frameCount].y : 0;
      console.log(positionsArr, frameCount);
      targetCtx.drawImage(
        faceImage,
        posX - faceImage.width / 2,
        posY - faceImage.height / 2,
        faceImage.width,
        faceImage.height
      );
    }
  }

  function playCanvas() {
    if (!playInterval)
      playInterval = setInterval(() => {
        setFrame(frameCount + 1);
      }, imagePlayRate);
  }

  function pauseCanvas() {
    if (playInterval) clearInterval(playInterval);
    playInterval = undefined;
  }

  function toggleCanvas() {
    if (!playInterval) playCanvas();
    else pauseCanvas();
  }

  function handleImageMouseDown(e) {
    canMouseX = parseInt(e.offsetX);
    canMouseY = parseInt(e.offsetY);
    const [posX, posY, actualPosX, actualPosY] = getCenterCoords(
      positionsArr[frameCount],
      faceImage
    );
    console.log(canMouseX, canMouseY, posX, posY);
    if (
      e.button === 0 &&
      canMouseX > posX &&
      canMouseY > posY &&
      canMouseX < posX + faceImage.width &&
      canMouseY < posY + faceImage.height
    ) {
      clickOrigPos = { x: canMouseX, y: canMouseY };
      imageOrigPos = { x: actualPosX, y: actualPosY };
      console.log("orig pos", clickOrigPos);
      isDragging = true;
    }
  }

  function handleImageMouseUp(e) {
    canMouseX = parseInt(e.offsetX);
    canMouseY = parseInt(e.offsetY);
    // clear the drag flag
    isDragging = false;
  }

  function handleImageMouseOut(e) {
    canMouseX = parseInt(e.offsetX);
    canMouseY = parseInt(e.offsetY);
    // user has left the canvas, so clear the drag flag
    isDragging = false;
  }

  function handleImageMouseMove(e) {
    canMouseX = parseInt(e.offsetX);
    canMouseY = parseInt(e.offsetY);
    // if the drag flag is set, clear the canvas and draw the image
    if (isDragging) {
      console.log(canMouseX, canMouseY);
      positionsArr[frameCount] = {
        x: imageOrigPos.x - (clickOrigPos.x - canMouseX),
        y: imageOrigPos.y - (clickOrigPos.y - canMouseY),
      };
      drawFrame();
    }
  }

  function drawFrame(frame = frameCount) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    drawAnimatedImage(frame);
    drawOverlayImage(frame);
  }
  function getCenterCoords(origPos, image) {
    const posX = origPos?.x ?? 0;
    const posY = origPos?.y ?? 0;
    const centerOffsetX = image.width / 2;
    const centerOffsetY = image.height / 2;
    return [posX - centerOffsetX, posY - centerOffsetY, posX, posY];
  }

  function generateGif() {
    const gif = new GIF({
      workers: 2,
      quality: 8,
      transparent: 0x000000,
      width: 128,
      height: 128,
    });
    console.log("POSITIONS ARR", positionsArr);
    const tempCanvas = document.createElement("canvas");
    tempCanvas.width = 128;
    tempCanvas.height = 128;
    const tempCtx = tempCanvas.getContext("2d");
    // tempCtx.fillStyle = "#ffffff";
    console.log("image arr", arr);
    arr.forEach((image, frameCount) => {
      tempCtx.clearRect(0, 0, tempCanvas.width, tempCanvas.height);
      //   tempCtx.fillRect(0, 0, tempCanvas.width, tempCanvas.height);
      console.log("frameCount", frameCount);
      drawAnimatedImage(frameCount, tempCtx);
      drawOverlayImage(frameCount, tempCtx);
      gif.addFrame(tempCtx, { copy: true, delay: imagePlayRate });
      console.log(image, frameCount);
    });
    gif.on("finished", function (blob) {
      console.log(blob);
      window.open(URL.createObjectURL(blob));
    });
    gif.render();

    console.log("test here");
  }
</script>

<div class="customize-canvas-container mdl-card mdl-shadow--2dp">
  <span class="gif-canvas-container">
    <canvas
      id="gifCanvas"
      class="gif-canvas"
      width="128"
      height="128"
      on:scroll={onScroll}
      on:mousedown={handleImageMouseDown}
      on:mousemove={handleImageMouseMove}
      on:mouseup={handleImageMouseUp}
      on:mouseout={handleImageMouseOut}
    /></span
  >
  <div class="frame-count-container">
    <button
      class="mdl-button mdl-js-button mdl-button--fab mdl-js-ripple-effect mdl-button--2mini-fab"
      on:click={() => setFrame(frameCount - 1)}
    >
      <i class="material-icons">remove</i>
    </button>
    <span class="mdl-chip mdl-chip--deletable">
      <span class="mdl-chip__text">Frame {frameCount + 1}</span>
      <button on:click={toggleCanvas} type="button" class="mdl-chip__action">
        {#if !playInterval}
          <i class="material-icons">play_arrow</i>
        {:else}
          <i class="material-icons">pause</i>
        {/if}
      </button>
    </span>
    <button
      class="mdl-button mdl-js-button mdl-button--fab mdl-js-ripple-effect mdl-button--2mini-fab"
      on:click={() => setFrame(frameCount + 1)}
    >
      <i class="material-icons">add</i>
    </button>
  </div>
  <div class="mdl-card__supporting-text">
    1. Press + to go forward, - to go backward<br>
    2. Drag the face around if its not positioned correctly <br>
  3. Press "Generate" to download!
  </div>
  <div class="mdl-card__actions mdl-card--border">
    <Button id="goback" onClick={goBackStep} buttonText="Back" />
    <Button id="generate" onClick={generateGif} buttonText="Generate" />
  </div>
</div>

<style>
  .customize-canvas-container {
    width: 100%;
  }

  .frame-count-container {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 200px;
    margin: auto;
  }

  .mdl-button--2mini-fab {
    height: 30px;
    min-width: 30px;
    width: 30px;
  }

  .gif-canvas {
    box-shadow: 0 2px 2px 0 rgb(0 0 0 / 14%), 0 3px 1px -2px rgb(0 0 0 / 20%),
      0 1px 5px 0 rgb(0 0 0 / 12%);
  }

  .gif-canvas-container {
    width: 128px;
    height: 128px;
    margin: 1em auto;
  }
</style>
