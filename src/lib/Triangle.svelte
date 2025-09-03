<script lang="ts" module>
  import { setContext, type Snippet } from "svelte";
  import {
    useOglContext,
    type MouseMoveCallback,
    type MountedOgl,
    type ResizeCallback,
    type UpdateCallback,
  } from "./Canvas.svelte";
  import type { AttributeMap as GeometryOptions } from "./types/core/Geometry.js";
  import { Triangle } from "ogl";

  export class OglTriangle {
    type = "triangle" as const;
    ogl: MountedOgl;
    id: string;
    triangle: Triangle;
    onMount?: (triangle: OglTriangle) => void;
    constructor(
      ogl: MountedOgl,
      id: string,
      triangleOptions?: GeometryOptions,
      om?: (triangle: OglTriangle) => void
    ) {
      this.ogl = ogl;
      this.id = id;
      this.triangle = new Triangle(ogl.gl, triangleOptions);
      setContext("triangle", this);
      ogl.addChild(this);
      om?.(this);
    }

    update(geometryOptions?: GeometryOptions) {
      if (!geometryOptions) return;
      // Store one VAO per program attribute locations order
      this.triangle.VAOs = {};
      this.triangle.drawRange = { start: 0, count: 0 };
      this.triangle.instancedCount = 0;
      // Unbind current VAO so that new buffers don't get added to active mesh
      this.ogl.gl!.renderer.bindVertexArray(null);
      this.ogl.gl!.renderer.currentGeometry = null;
      this.triangle.attributes = geometryOptions;
      for (let key in geometryOptions) {
        this.triangle.addAttribute(key, geometryOptions[key]);
        const attr = this.triangle.attributes[key];
        console.log(attr);
      }
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
    attributes: triangleOptions,
    children,
    onMount: om,
  }: {
    id?: string;
    onResize?: ResizeCallback<OglTriangle>;
    onUpdate?: UpdateCallback<OglTriangle>;
    onMouseMove?: MouseMoveCallback<OglTriangle>;
    onMouseLeave?: MouseMoveCallback<OglTriangle>;
    onMouseEnter?: MouseMoveCallback<OglTriangle>;
    onPointerDown?: MouseMoveCallback<OglTriangle>;
    onPointerUp?: MouseMoveCallback<OglTriangle>;
    attributes?: GeometryOptions;
    children?: Snippet<[OglTriangle]>;
    onMount?: (triangle: OglTriangle) => void;
  } = $props();
  const ogl = useOglContext();
  let id = $props.id();
  const triangle = new OglTriangle(ogl, customId || id, triangleOptions, om);

  ogl.useEvent("resize", triangle, onResize);
  ogl.useEvent("update", triangle, onUpdate);
  ogl.useEvent("mousemove", triangle, onMouseMove);
  ogl.useEvent("mouseleave", triangle, onMouseLeave);
  ogl.useEvent("mouseenter", triangle, onMouseEnter);
  ogl.useEvent("pointerdown", triangle, onPointerDown);
  ogl.useEvent("pointerup", triangle, onPointerUp);

  $effect(() => {
    if (triangleOptions) {
      triangle.update(triangleOptions);
    }
  });
</script>

{@render children?.(triangle)}
