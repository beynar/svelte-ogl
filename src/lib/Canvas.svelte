<script lang="ts" module>
  import { Color, Renderer } from "ogl";
  import type { RendererOptions } from "./types/core/Renderer.js";
  import {
    getContext,
    onDestroy,
    onMount,
    setContext,
    untrack,
    type Snippet,
  } from "svelte";
  import { on } from "svelte/events";

  type EventType =
    | "resize"
    | "update"
    | "mousemove"
    | "pointerdown"
    | "pointerup"
    | "mouseenter"
    | "mouseleave";

  type MousePayload = { x: number; y: number; rect: DOMRect; e: MouseEvent };
  type EventPayloads = {
    resize: { width: number; height: number };
    update: { time: number };
    mousemove: MousePayload;
    pointerdown: MousePayload;
    pointerup: MousePayload;
    mouseenter: MousePayload;
    mouseleave: MousePayload;
  };

  type Handler<T extends EventType, C> = (
    payload: EventPayloads[T],
    ctx: C
  ) => void;

  export type ResizeCallback<T> = (
    { width, height }: { width: number; height: number },
    context: T
  ) => void;
  export type UpdateCallback<T> = (
    { time }: { time: number },
    context: T
  ) => void;
  type OglContextOptions = {
    onMount?: (context: MountedOgl) => void;
    dpr?: number;
    canvasClassName?: string;
  };
  type MakeRequired<T, K extends keyof T> = Omit<T, K> & {
    [P in K]: T[P] extends infer U | null ? U : T[P];
  };
  export type MountedOgl = MakeRequired<
    OglContext,
    "gl" | "container" | "canvas" | "renderer"
  >;

  export type MouseMoveCallback<T> = (
    { x, y, rect, e }: { x: number; y: number; rect: DOMRect; e: MouseEvent },
    context: T
  ) => void;

  export type PointerDownCallback<T> = (
    { x, y, rect, e }: { x: number; y: number; rect: DOMRect; e: MouseEvent },
    context: T
  ) => void;
  export type PointerUpCallback<T> = (
    { x, y, rect, e }: { x: number; y: number; rect: DOMRect; e: MouseEvent },
    context: T
  ) => void;
  export type MouseEnterCallback<T> = (
    { x, y, rect, e }: { x: number; y: number; rect: DOMRect; e: MouseEvent },
    context: T
  ) => void;
  export type MouseLeaveCallback<T> = (
    { x, y, rect, e }: { x: number; y: number; rect: DOMRect; e: MouseEvent },
    context: T
  ) => void;

  export class OglContext {
    private options: OglContextOptions;
    mousePosition = $state<[number, number]>([0, 0]);
    containerSize = $state<[number, number]>([0, 0]);
    mounted = $state(false);
    renderer: Renderer | null = null;
    container: HTMLDivElement | null = null;
    canvas: HTMLCanvasElement | null = null;
    gl: Renderer["gl"] | null = null;
    raf: number | null = null;
    children: Map<string, any> = new Map();
    private listeners = new Map<EventType, Set<Function>>();

    onResize = () => {
      if (!this.container) return;
      const { width, height } = this.container.getBoundingClientRect();

      this.renderer?.setSize(width, height);

      this.emit("resize", { width, height });

      if (this.canvas) {
        this.canvas.style.width = "100%";
        this.canvas.style.height = "100%";
      }
    };

    onUpdate = (time: number) => {
      this.raf = requestAnimationFrame(this.onUpdate);
      this.emit("update", { time });
    };

    handleMouseEvent = (e: MouseEvent, eventType: EventType) => {
      if (!this.container || !this.mounted) {
        return;
      }
      const rect = this.container.getBoundingClientRect();
      const x = (e.clientX - rect.left) / rect.width;
      const y = 1.0 - (e.clientY - rect.top) / rect.height;
      this.mousePosition = [x, y];
      this.emit(eventType, { x, y, rect, e });
    };

    mouseEvent = (type: EventType) => (event: MouseEvent) => {
      this.handleMouseEvent(event, type);
    };

    on<T extends EventType, C>(type: T, ctx: C, handler: Handler<T, C>) {
      let set = this.listeners.get(type);
      if (!set) this.listeners.set(type, (set = new Set()));
      const wrapped = (p: EventPayloads[T]) => handler(p, ctx);
      set.add(wrapped);
      return () => set!.delete(wrapped);
    }

    emit<T extends EventType>(type: T, payload: EventPayloads[T]) {
      const set = this.listeners.get(type);
      if (!set || !set.size) return;
      for (const h of set) (h as (p: EventPayloads[T]) => void)(payload);
    }

    useEvent<T extends EventType, C>(type: T, ctx: C, handler?: Handler<T, C>) {
      if (!handler) return () => {};
      const off = this.on(type, ctx, handler);
      return onDestroy(off);
    }

    addChild = (child: any) => {
      this.children.set(child.id, child);
      onMount(() => {
        return () => {
          this.children.delete(child.id);
        };
      });
    };

    attachment(
      node: HTMLDivElement,
      rendererOptions: Partial<RendererOptions>
    ) {
      this.container = node;
      const resizeObserver = new ResizeObserver((entries) => {
        const { width, height } = entries[0].contentRect;
        this.containerSize = [width, height];
        this.onResize();
      });
      resizeObserver.observe(node);
      this.containerSize = [node.clientWidth, node.clientHeight];

      this.renderer = new Renderer({
        alpha: true,
        premultipliedAlpha: true,
        antialias: true,
        ...rendererOptions,
      });
      this.gl = this.renderer.gl;
      this.canvas = this.renderer.gl.canvas;
      this.container.appendChild(this.canvas!);
      this.canvas!.style.width = "100%";
      this.canvas!.style.height = "100%";
      this.canvas!.style.display = "block";
      if (this.options.canvasClassName) {
        this.canvas!.className = this.options.canvasClassName;
      }

      this.mounted = true;
      this.options.onMount?.(this as MountedOgl);
      this.raf = requestAnimationFrame(this.onUpdate);

      return () => {
        this.mounted = false;
        this.renderer?.gl.canvas.remove();
        if (
          this.container &&
          this.gl &&
          this.gl.canvas.parentNode === this.container
        ) {
          this.container.removeChild(this.gl.canvas);
        }
        this.gl?.getExtension("WEBGL_lose_context")?.loseContext();
        this.renderer = null;
        this.container = null;
        this.canvas = null;
        this.gl = null;
        if (this.raf) cancelAnimationFrame(this.raf);
        this.raf = null;
        this.listeners.clear();
      };
    }

    constructor(options: OglContextOptions) {
      this.options = options;
      setContext("ogl", this);
    }

    // Colors utils
    color = {
      hexToArray: (hex: string): [number, number, number] => {
        const color = new Color(hex);
        return [color.r, color.g, color.b] as [number, number, number];
      },
    };
  }

  export const useOglContext = () => {
    const oglContext = getContext<MountedOgl>("ogl");
    if (!oglContext) throw new Error("No OglContext found");
    return oglContext;
  };
</script>

<script lang="ts">
  let {
    class: className = "",
    children,
    onResize,
    onUpdate,
    onMouseMove,
    onMount: om,
    canvasClassName,
    ogl = $bindable(),
    onMouseLeave,
    onMouseEnter,
    onPointerDown,
    onPointerUp,
    ...rendererOptions
  }: {
    class?: string;
    children?: Snippet<[MountedOgl]>;
    onResize?: ResizeCallback<OglContext>;
    onUpdate?: UpdateCallback<OglContext>;
    onMouseMove?: MouseMoveCallback<OglContext>;
    onMount?: (context: MountedOgl) => void;
    canvasClassName?: string;
    ogl?: OglContext | null;
    onMouseLeave?: MouseMoveCallback<OglContext>;
    onMouseEnter?: MouseMoveCallback<OglContext>;
    onPointerDown?: MouseMoveCallback<OglContext>;
    onPointerUp?: MouseMoveCallback<OglContext>;
  } & Partial<Omit<RendererOptions, "canvas">> = $props();

  const oglContext = new OglContext({
    onMount: (ogl) => {
      om?.(ogl);
      ogl.onResize();
    },
    canvasClassName,
  });

  oglContext.useEvent("resize", oglContext, onResize);
  oglContext.useEvent("update", oglContext, onUpdate);
  oglContext.useEvent("mousemove", oglContext, onMouseMove);
  oglContext.useEvent("mouseleave", oglContext, onMouseLeave);
  oglContext.useEvent("mouseenter", oglContext, onMouseEnter);
  oglContext.useEvent("pointerdown", oglContext, onPointerDown);
  oglContext.useEvent("pointerup", oglContext, onPointerUp);

  ogl = oglContext;
</script>

<div
  role="application"
  onmousemove={oglContext.mouseEvent("mousemove")}
  onmouseleave={oglContext.mouseEvent("mouseleave")}
  onmouseenter={oglContext.mouseEvent("mouseenter")}
  onpointerdown={oglContext.mouseEvent("pointerdown")}
  class={className}
  {@attach (node) => oglContext.attachment(node, rendererOptions)}
  style:width="100%"
  style:max-width="100%"
  style:overflow="hidden"
  style:height="100%"
  style:display="block"
>
  {#if oglContext.mounted}
    {@render children?.(oglContext as MountedOgl)}
  {/if}
</div>
