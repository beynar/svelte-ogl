<script lang="ts" module>
  import { getContext, onMount, setContext, type Snippet } from "svelte";
  import {
    useOglContext,
    type MouseMoveCallback,
    type MountedOgl,
    type ResizeCallback,
    type UpdateCallback,
  } from "./Canvas.svelte";
  import { Mesh } from "ogl";
  import type { MeshOptions } from "./types/core/Mesh.js";
  import { OglProgram } from "./Program.svelte";
  import type { OglGeometry } from "./Geometry.svelte";

  type PartialMeshOptions = Omit<MeshOptions, "program" | "geometry"> & {
    program?: OglProgram["program"];
    geometry?: OglGeometry["geometry"];
  };
  export class OglMesh {
    type = "mesh" as const;
    ogl: MountedOgl;
    mesh: Mesh;
    id: string;
    constructor(
      ogl: MountedOgl,
      id: string,
      meshOptions: PartialMeshOptions,
      om?: (context: OglMesh) => void
    ) {
      this.ogl = ogl;
      this.id = id;
      const program =
        meshOptions.program || getContext<OglProgram>("program")?.program;
      const geometry =
        meshOptions.geometry ||
        getContext<any>("geometry")?.geometry ||
        getContext<any>("triangle")?.triangle;
      this.mesh = new Mesh(ogl.gl!, { ...meshOptions, program, geometry });
      setContext("mesh", this);
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
    ...meshOptions
  }: {
    id?: string;
    onResize?: ResizeCallback<OglMesh>;
    onUpdate?: UpdateCallback<OglMesh>;
    onMouseMove?: MouseMoveCallback<OglMesh>;
    onMouseLeave?: MouseMoveCallback<OglMesh>;
    onMouseEnter?: MouseMoveCallback<OglMesh>;
    onPointerDown?: MouseMoveCallback<OglMesh>;
    onPointerUp?: MouseMoveCallback<OglMesh>;
    onMount?: (context: OglMesh) => void;
    children?: Snippet<[OglMesh]>;
  } & PartialMeshOptions = $props();
  const ogl = useOglContext();
  let id = $props.id();
  const mesh = new OglMesh(ogl, customId || id, meshOptions, om);

  ogl.useEvent("resize", mesh, onResize);
  ogl.useEvent("update", mesh, onUpdate);
  ogl.useEvent("mousemove", mesh, onMouseMove);
  ogl.useEvent("mouseleave", mesh, onMouseLeave);
  ogl.useEvent("mouseenter", mesh, onMouseEnter);
  ogl.useEvent("pointerdown", mesh, onPointerDown);
  ogl.useEvent("pointerup", mesh, onPointerUp);

  $effect(() => {
    Object.keys(meshOptions).forEach((key) => {
      Object.assign(
        mesh.mesh,
        key,
        meshOptions[key as keyof typeof meshOptions]
      );
    });
  });
</script>

{@render children?.(mesh)}
