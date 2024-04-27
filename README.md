# DXGraphicsAPI
<!--- A 3D Graphics engine/API for DirectX11.0 supporting lighting, terrain, per-mesh rendering and more -->
![Graphics Scene](/images/FinalScene.png)
## Overview
This project is a graphics API for DirectX 11.0 that makes the Direct3D interface easier to use. <br/>

Features include:
- Phong Lighting system with directional, point, and spot lights
- Simplified Rendering (GraphicsObjects)
- Custom terrain with heightmap images
- Import custom model from file
- Shader classes to handle texture, light, and color data
- Mirror and fog effects

## Graphics Objects
One of the features I implemented is a GraphicsObject class that performed dealing with shader, texture, and model data under one centralized render call: 
```C++
void GraphicsObject_TextureLight::Render() {
	//Set vertex and index buffer to context through Model
	pModel->SetToContext(pShader->GetContext());

	//Per mesh, connect the corresponding texture, world matrix, and material properties
	for (int i = 0; i < pModel->GetMeshCount(); i++) {
		pShader->SetTexture(MeshTextures[i]);
		pShader->SendWorldAndMaterial(World, MeshMatAmbient[i], MeshMatDiffuse[i], MeshMatSpecular[i]);
		pModel->RenderMesh(pShader->GetContext(), i);
	}
}
```
<br>
This simplified the draw stage because users would place all necessary data during initialization into the Graphics Object, and during the Render() call, data such as per-mesh material properties or textures would just be connected to the shader to activate the pipeline.

<br/> The Graphics Object had multiple derived classes for each type of shader, making it easier to automatically connect the necessary shader-specific data during rendering: <br/>

![GraphicsObject](/images/GraphicsObject.png)


## Mirror
Another feature I implemented is a mirror effect, where the illusion of reflection is given by rendering objects from one side of the mirror twice. <br/>

Youtube Demo: <br/>
[![MirrorDemo](https://img.youtube.com/vi/eR4eGSRtDbU/0.jpg)](https://www.youtube.com/watch?v=eR4eGSRtDbU) <br>

While costly when used for a real scene, this effect got me to be familiar with using the stencil buffer for multiple renders. To do this effect, I created a special mirror class that would handle different rasterizer states, blend states, and depth stencil states.
Here is a function that a user would call to render the mirror object into a stencil buffer to use later:
```C++
void Mirror::WriteMirrorToStencil(ID3D11DeviceContext* contextPtr, const Camera& cam) {
	//*//Render mirror without writing to render target and write to stencil buffer

	// BLEND STATE: Stop writing to the render target 
	contextPtr->OMSetBlendState(NoWriteToRenderTargetBS, nullptr, 0xffffffff);
	// STENCIL: Set up the stencil for marking ('1' for all pixels that passed the depth test. See comment at line 35)
	contextPtr->OMSetDepthStencilState(MarkMirrorDSS, 1);

	// Set Color Shader to context
	pShader->SetToContext(contextPtr);
	pShader->SendCamMatrices(cam.getViewMatrix(), cam.getProjMatrix());
	// Render the mirror object
	pShader->SendWorldColor(worldMat, Colors::DarkGray); // The color is irrelevant here
	pPlane->Render(contextPtr);

	// STENCIL: stop using the stencil
	contextPtr->OMSetDepthStencilState(0, 0);
	// BLEND STATE: Return the blend state to normal (writing to render target)
	contextPtr->OMSetBlendState(0, nullptr, 0xffffffff);


	//////Prepare Rasterizer and depth stencil state for rendering "reflected" objects
	
	// WINDINGS: face winding will appear inside out after reflection. Switching to CW front facing
	contextPtr->RSSetState(MirrorFrontFaceAsClockWiseRS);
	// STENCIL: Use the stencil test (reference value 1) and only pass the test if the stencil already had a one present
	contextPtr->OMSetDepthStencilState(DrawReflectionDSS, 1);
}
```


