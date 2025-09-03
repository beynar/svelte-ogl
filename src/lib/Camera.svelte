<script lang="ts" module>
  import { setContext, type Snippet } from "svelte";
  import {
    useOglContext,
    type MouseMoveCallback,
    type MountedOgl,
    type ResizeCallback,
    type UpdateCallback,
  } from "./Canvas.svelte";
  import type { CameraOptions } from "./types/core/Camera.js";
  import { Camera } from "ogl";

  export class OglCamera {
    type = "camera" as const;
    ogl: MountedOgl;
    id: string;
    camera: Camera;
    constructor(
      ogl: MountedOgl,
      id: string,
      cameraOptions?: Partial<CameraOptions>,
      om?: (context: OglCamera) => void
    ) {
      this.ogl = ogl;
      this.id = id;
      this.camera = new Camera(ogl.gl, cameraOptions);
      setContext("camera", this);
      ogl.addChild(this);
      om?.(this);
    }
  }
</script>

<script lang="ts">
  let {
    id: customId,
    onResize,
    onUpdate,
    onMouseMove,
    onMouseLeave,
    onMouseEnter,
    onPointerDown,
    onPointerUp,
    onMount: om,
    children,
    ...cameraOptions
  }: {
    id?: string;
    onResize?: ResizeCallback<OglCamera>;
    onUpdate?: UpdateCallback<OglCamera>;
    onMouseMove?: MouseMoveCallback<OglCamera>;
    onMouseLeave?: MouseMoveCallback<OglCamera>;
    onMouseEnter?: MouseMoveCallback<OglCamera>;
    onPointerDown?: MouseMoveCallback<OglCamera>;
    onPointerUp?: MouseMoveCallback<OglCamera>;
    onMount?: (context: OglCamera) => void;
    children?: Snippet<[OglCamera]>;
  } & Partial<CameraOptions> = $props();
  const ogl = useOglContext();
  let id = $props.id();
  const camera = new OglCamera(ogl, customId || id, cameraOptions, om);

  ogl.useEvent("resize", camera, onResize);
  ogl.useEvent("update", camera, onUpdate);
  ogl.useEvent("mousemove", camera, onMouseMove);
  ogl.useEvent("mouseleave", camera, onMouseLeave);
  ogl.useEvent("mouseenter", camera, onMouseEnter);
  ogl.useEvent("pointerdown", camera, onPointerDown);
  ogl.useEvent("pointerup", camera, onPointerUp);

  $effect(() => {
    Object.keys(cameraOptions || {}).forEach((key) => {
      Object.assign(camera.camera, {
        [key]: cameraOptions[key as keyof typeof cameraOptions],
      });
    });
  });
</script>

{@render children?.(camera)}
