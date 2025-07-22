<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  // Props del componente
  export let basePath: string = "http://127.0.0.1:12393/live2d-models/shizuku/"; 
  export let modelPath: string = basePath + "shizuku.model.json";
  export let width: number = 400;
  export let height: number = 600;

  let canvas: HTMLCanvasElement;
  let gl: WebGLRenderingContext | null = null;
  let live2dModel: any = null;
  let isLoaded: boolean = false;
  let error: string = "";
  let animationId: number = 0;

  onMount(async () => {
    try {
      await initializeLive2D();
    } catch (err: any) {
      error = `Error: ${err.message}`;
      console.error(error, err);
    }
  });

  onDestroy(() => {
    if (animationId) {
      cancelAnimationFrame(animationId);
    }
    if (canvas) {
      canvas.removeEventListener('mousemove', handleMouseMove);
      canvas.removeEventListener('click', handleClick);
    }
  });

  async function initializeLive2D() {
    if (!canvas) return;

    const Live2D = (window as any).Live2D;
    const Live2DModelWebGL = (window as any).Live2DModelWebGL;
    
    if (!Live2D || !Live2DModelWebGL) {
      throw new Error("Live2D debe estar cargado");
    }

    Live2D.init();

    // Obtener contexto WebGL directamente
    gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl') as WebGLRenderingContext;
    if (!gl) {
      throw new Error("WebGL no está soportado");
    }

    // Configurar viewport
    gl.viewport(0, 0, width, height);
    gl.clearColor(0.0, 0.0, 0.0, 0.0);

    // Cargar modelo JSON
    const modelJsonResponse = await fetch(modelPath);
    if (!modelJsonResponse.ok) throw new Error(`No se pudo cargar: ${modelPath}`);
    
    const modelJson = await modelJsonResponse.json();
    console.log("Modelo JSON cargado:", modelJson);

    // Cargar archivo MOC binario
    const mocPath = basePath + modelJson.model;
    console.log("Cargando archivo MOC desde:", mocPath);
    
    const mocResponse = await fetch(mocPath);
    if (!mocResponse.ok) throw new Error(`No se pudo cargar el archivo MOC: ${mocPath}`);
    
    // Obtener los datos binarios como ArrayBuffer
    const mocArrayBuffer = await mocResponse.arrayBuffer();
    console.log("Archivo MOC cargado, tamaño:", mocArrayBuffer.byteLength, "bytes");

    // Crear modelo usando los datos binarios
    live2dModel = Live2DModelWebGL.loadModel(mocArrayBuffer);
    if (!live2dModel) {
      throw new Error('No se pudo crear el modelo Live2D');
    }

    console.log("Modelo Live2D creado exitosamente");

    // Cargar texturas usando WebGL directamente
    if (modelJson.textures?.length > 0) {
      console.log("Cargando texturas:", modelJson.textures);
      
      for (let i = 0; i < modelJson.textures.length; i++) {
        const textureFile = modelJson.textures[i];
        const textureUrl = basePath + textureFile;
        console.log("Cargando textura:", textureUrl);
        
        try {
          const texture = await loadWebGLTexture(gl, textureUrl);
          if (texture) {
            live2dModel.setTexture(i, texture);
          }
        } catch (textureError) {
          console.warn(`Error cargando textura ${i}:`, textureError);
        }
      }
    }

    // Configurar posición y escala del modelo
    const modelWidth = live2dModel.getCanvasWidth();
    const modelHeight = live2dModel.getCanvasHeight();
    
    console.log(`Dimensiones del modelo: ${modelWidth}x${modelHeight}`);
    
    // Calcular escala para que el modelo encaje en el canvas
    const scaleX = (width * 0.9) / modelWidth;
    const scaleY = (height * 0.9) / modelHeight;
    const scale = Math.max(scaleX, scaleY);
    
    // Centrar el modelo
    const offsetX = width / 2;
    const offsetY = height / 2;
    
    // Aplicar transformaciones
    live2dModel.setMatrix(
      1, 0, 0, 1, 
      offsetX, offsetY
    );

    console.log(`Modelo configurado - Escala: ${scale}, Offset: ${offsetX}, ${offsetY}`);

    // Configurar parámetros iniciales
    live2dModel.setParamFloat('PARAM_EYE_L_OPEN', 1);
    live2dModel.setParamFloat('PARAM_EYE_R_OPEN', 1);
    live2dModel.setParamFloat('PARAM_MOUTH_OPEN_Y', 0);
    live2dModel.setParamFloat('PARAM_BREATH', 0.5);

    // Configurar eventos e iniciar animación
    setupInteraction();
    startAnimation();
    
    isLoaded = true;
    console.log("Live2D modelo cargado exitosamente");
  }

  async function loadWebGLTexture(gl: WebGLRenderingContext, url: string): Promise<WebGLTexture | null> {
    return new Promise((resolve, reject) => {
      const image = new Image();
      image.crossOrigin = "anonymous";
      
      image.onload = () => {
        const texture = gl.createTexture();
        if (!texture) {
          reject(new Error('No se pudo crear la textura'));
          return;
        }

        gl.bindTexture(gl.TEXTURE_2D, texture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);
        
        // Configurar filtros de textura
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
        gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
        
        resolve(texture);
      };
      
      image.onerror = () => reject(new Error(`No se pudo cargar la imagen: ${url}`));
      image.src = url;
    });
  }

  function setupInteraction() {
    if (!canvas) return;

    canvas.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('click', handleClick);
  }

  function startAnimation() {
    if (!gl || !live2dModel) return;

    const animate = () => {
      if (!gl || !live2dModel || !isLoaded) return;

      // Limpiar el canvas
      gl.clear(gl.COLOR_BUFFER_BIT);

      // Animaciones automáticas
      const t = Date.now() * 0.001;
      
      // Respiración
      const breathValue = (Math.sin(t * 2) + 1) * 0.5;
      live2dModel.setParamFloat('PARAM_BREATH', breathValue);
      
      // Movimiento sutil de la cabeza
      live2dModel.setParamFloat('PARAM_ANGLE_Z', Math.sin(t * 0.3) * 3);
      
      // Parpadeo ocasional
      if (Math.random() < 0.005) { // 0.5% probabilidad por frame
        live2dModel.setParamFloat('PARAM_EYE_L_OPEN', 0);
        live2dModel.setParamFloat('PARAM_EYE_R_OPEN', 0);
        setTimeout(() => {
          if (live2dModel) {
            live2dModel.setParamFloat('PARAM_EYE_L_OPEN', 1);
            live2dModel.setParamFloat('PARAM_EYE_R_OPEN', 1);
          }
        }, 150);
      }

      // Actualizar y dibujar el modelo
      live2dModel.update();
      live2dModel.draw(gl);

      animationId = requestAnimationFrame(animate);
    };

    animate();
  }

  function handleMouseMove(event: MouseEvent) {
    if (!live2dModel || !canvas) return;
    
    const rect = canvas.getBoundingClientRect();
    const mouseX = (event.clientX - rect.left - width / 2) / (width / 2);
    const mouseY = (event.clientY - rect.top - height / 2) / (height / 2);

    // Movimiento de cabeza y ojos suave
    live2dModel.setParamFloat("PARAM_ANGLE_X", mouseX * 15);
    live2dModel.setParamFloat("PARAM_ANGLE_Y", mouseY * 10);
    live2dModel.setParamFloat("PARAM_EYE_BALL_X", mouseX * 0.7);
    live2dModel.setParamFloat("PARAM_EYE_BALL_Y", mouseY * 0.7);
  }

  function handleClick() {
    if (!live2dModel || !isLoaded) return;
    
    // Animación de sonrisa al hacer clic
    live2dModel.setParamFloat('PARAM_MOUTH_FORM', 1);
    live2dModel.setParamFloat('PARAM_MOUTH_OPEN_Y', 0.3);
    
    setTimeout(() => {
      if (live2dModel) {
        live2dModel.setParamFloat('PARAM_MOUTH_FORM', 0);
        live2dModel.setParamFloat('PARAM_MOUTH_OPEN_Y', 0);
      }
    }, 800);
  }
</script>

<div class="live2d-container">
  <canvas
    bind:this={canvas}
    {width}
    {height}
    class="live2d-canvas"
    class:loaded={isLoaded}
  />
  
  {#if error}
    <div class="error-overlay">
      <p>❌ {error}</p>
    </div>
  {/if}
  
  {#if !isLoaded && !error}
    <div class="loading-overlay">
      <p>🔄 Cargando modelo Live2D...</p>
    </div>
  {/if}
</div>

<style>
  .live2d-container {
    position: relative;
    display: inline-block;
  }
  
  .live2d-canvas {
    border: 1px solid #ddd;
    border-radius: 8px;
    background: transparent;
    transition: opacity 0.3s;
  }
  
  .live2d-canvas.loaded {
    cursor: pointer;
  }
  
  .error-overlay, .loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 14px;
    text-align: center;
    padding: 20px;
  }
  
  .error-overlay {
    background: rgba(255, 0, 0, 0.1);
    color: #ff4444;
  }
  
  .loading-overlay {
    background: rgba(0, 0, 0, 0.1);
    color: #666;
  }
</style>