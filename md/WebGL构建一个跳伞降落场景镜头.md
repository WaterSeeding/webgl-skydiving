# WebGL构建一个跳伞降落场景镜头

> 这里我网上找到一个WebGL开源小Demo，作者仅通过一张天空盒、一个人物模型，就构建了这么个精彩动画的场景，真的很赞！
>
> 于是，抱着学习的态度，我整理了以下开发的思路，方便自己借鉴、学习。
> 
> - [查看地址](https://webgl-skydiving.vercel.app/)
> - [仓库地址](https://github.com/sebastien-lempens/webgl-skydiving)

<br />

## 介绍

![降落效果](./WebGL构建一个跳伞降落场景镜头/1.gif)

如果所示，一个勇敢的少年，背着还未打开降落伞，投身如广袤的天地中，在重力的作用下，风都如何袖带般，在他身边穿梭而过，簌簌地声响肉眼可见，那强大的冲击力，震荡着整个天地都在晃荡~~

<br />

## 思路

> 好了以下，从代码角度介绍，作者开发历程：

![降落效果](./WebGL构建一个跳伞降落场景镜头/1.png)

1. 创建全景天空盒：

```tsx
import { useTexture } from "@react-three/drei";
import { Sphere } from "@react-three/drei";
import { BackSide } from "three";
export const World = () => {
  const skyTexture = useTexture("sky-texture.jpg");
  return (
    <Sphere args={[5, 60, 60]}>
      <meshBasicMaterial toneMapped={false} map={skyTexture} side={BackSide} />
    </Sphere>
  );
};
```

这里加载`skyTexture`设置`meshBasicMaterial.map`属性，构建全景天空盒。

![降落效果](./WebGL构建一个跳伞降落场景镜头/2.gif)

<br />

2. 添加少年人物模型：

```tsx
import { useEffect } from "react";
import { useFrame } from "@react-three/fiber";
import { useAnimations, useGLTF, useTexture } from "@react-three/drei";
import { DoubleSide } from "three";
export const Model = () => {
  const [skyDiverTextureBaseColor, skyDiverTextureRoughness, skyDiverTextureMetallic, skyDiverTextureNormal, skyDiverTextureClothes] =
    useTexture(
      [
        "texture/skydiver_BaseColor.webp",
        "texture/skydiver_Roughness.webp",
        "texture/skydiver_Metallic.webp",
        "texture/skydiver_Normal.webp",
        "texture/skydiver_Clothes.webp",
      ],
      ([baseColor, roughness, metallic, normal, clothes]) => {
        baseColor.flipY = roughness.flipY = metallic.flipY = normal.flipY = clothes.flipY = false;
      }
    );
  const test = useGLTF("/skydiver.glb");
  const { nodes, animations, scene } = useGLTF("/skydiver.glb");
  const { ref, actions, names } = useAnimations(animations, scene);
  const { mixamorigHips, skydiver_2: skydiver } = nodes;
  const onBeforeCompile = shader => {
    Object.assign(shader.uniforms, { ...skydiver.material.uniforms });
    shader.vertexShader = `
        uniform float uTime;
        uniform sampler2D uClothes;
        ${shader.vertexShader}
        `;
    shader.vertexShader = shader.vertexShader.replace(
      `#include <begin_vertex>`,
      `
          vec3 clothesTexture = vec3(texture2D(uClothes, vUv));
          float circleTime = 2.0;
          float amplitude = 30.0;
          float circleTimeParam = mod(uTime, circleTime);
          vec3 transformed = vec3( position );
          transformed.y += min(clothesTexture.y * sin( circleTimeParam * amplitude * (PI  / circleTime)) * 0.025, 0.5);
        `
    );
  };
  useEffect(() => {
    actions["animation_0"].reset().play();
    skydiver.material.uniforms = {
      uTime: { value: 0 },
      uClothes: { value: skyDiverTextureClothes },
    };
  }, []);
  useFrame(({ clock }) => {
    if (skydiver.material.uniforms?.uTime) {
      skydiver.material.uniforms.uTime.value = clock.getElapsedTime();
    }
  });
  return (
    <group dispose={null}>
      <group ref={ref}>
        <primitive object={mixamorigHips} />
        <skinnedMesh geometry={skydiver.geometry} skeleton={skydiver.skeleton}>
          <meshStandardMaterial
            side={DoubleSide}
            map={skyDiverTextureBaseColor}
            roughnessMap={skyDiverTextureRoughness}
            metalnessMap={skyDiverTextureMetallic}
            normalMap={skyDiverTextureNormal}
            normalScale={[-0.2, 0.2]}
            envMapIntensity={0.8}
            toneMapped={false}
            onBeforeCompile={onBeforeCompile}
            uniforms={{ uTime: { value: 0 } }}
          />
        </skinnedMesh>
      </group>
    </group>
  );
};

```

这里添加人物模型，并通过`meshStandardMaterial.onBeforeCompile`属性添加`Shader`效果，设置衣物动态效果。

![降落效果](./WebGL构建一个跳伞降落场景镜头/3.gif)

<br />

3. 添加袖带般的可视化风：

```tsx
import { useRef } from "react";
import { useFrame, useThree } from "@react-three/fiber";
import { Instances, Instance } from "@react-three/drei";
import { AdditiveBlending, DoubleSide, MathUtils } from "three";
import { Vector3 } from "three";

function WindShape() {
  const ref = useRef();
  const state = useThree();
  const { height: viewPortHeight } = state.viewport.getCurrentViewport();
  const v3 = new Vector3();
  const randomPosition = {
    x: MathUtils.randFloatSpread(8),
    y: MathUtils.randFloatSpread(5),
    z: MathUtils.randFloatSpread(8),
  };
  const randomSpeed = MathUtils.randFloat(0.05, 0.5);
  useFrame(({ camera }) => {
    if (ref.current) {
      const { current: el } = ref;
      const { height: elHeight } = el.instance.current.geometry.parameters;
      const { y: elPosition } = el.position;
      const worldPosition = el.getWorldPosition(v3);
      const limitPos = viewPortHeight - (worldPosition.y + elHeight / 2);
      if (limitPos < 0) {
        el.position.y = -(viewPortHeight + elHeight / 2);
      }
      el.position.y += randomSpeed;
      el.rotation.y = camera.rotation.y;
    }
  });
  return <Instance ref={ref} color='white' position={[randomPosition.x, randomPosition.y, randomPosition.z]} />;
}

export const WindEffect = () => {
  const INSTANCE = {
    number: 130,
  };
  return (
    <group>
      <Instances>
        <planeGeometry args={[0.0135, 1.2]} />
        <meshBasicMaterial side={DoubleSide} blending={AdditiveBlending} opacity={0.15} transparent />
        {Array(INSTANCE.number)
          .fill()
          .map((_, key) => (
            <WindShape key={key} />
          ))}
      </Instances>
    </group>
  );
};
```

这里添加`planeGeometry`作为袖带`instance`，通过`useFrame`函数变化`instance.position`。

![降落效果](./WebGL构建一个跳伞降落场景镜头/4.gif)
