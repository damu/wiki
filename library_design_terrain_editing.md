# Library Design: Improving Terrain Editing in Urho3D

[Unfinished]

Urho3D is a 3D game engine. It has a terrain system based on heightmaps and splatting maps. A splatting map is a texture whose colors say where to blend which other texture with what strength.

The height of the terrain can only be changed by manipulating the height map. This can be done like this:
```C++
IntVector2 v=terrain->WorldToHeightMap(cameraNode_->GetWorldPosition());
Image* i=terrain->GetHeightMap();
for(int x=-10;x<10;x++)
    for(int y=-10;y<10;y++)
        i->SetPixel(v.x_+x,v.y_+y,i->GetPixel(v.x_+x,v.y_+y)+Color(0.1,0.1,0.1));
terrain->ApplyHeightMap();
```

The splatting map texture can be changed with:
```C++
IntVector2 v=terrain->WorldToHeightMap(cameraNode_->GetWorldPosition());
// TU_DIFFUSE is defined to be 0, as in "the first texture" which is the splatting map 
// in the default terrain material
Texture2D* t=(Texture2D*)terrain->GetMaterial()->GetTexture(TU_DIFFUSE);
uint32_t c=Color(1,0,0).ToUInt();
for(int x=-10;x<10;x++)
    for(int y=-10;y<10;y++)
        t->SetData(0,v.x_+x,v.y_+y,1,1,&c);
terrain->GetMaterial()->SetTexture(TU_DIFFUSE,t);
```

```C++
IntVector2 v=terrain->WorldToHeightMap(cameraNode_->GetWorldPosition());
HeightMapModifier hm=terrain->GetHeightMapModifier();
for(int x=-10;x<10;x++)
    for(int y=-10;y<10;y++)
        hm.Pixel(v.x_+x,v.y_+y)+=0.1;
```

```C++
IntVector2 v=terrain->WorldToHeightMap(cameraNode_->GetWorldPosition());
SplattingMapModifier sm=terrain->GetSplattingMapModifier();
uint32_t c=Color(1,0,0).ToUInt();
for(int x=-10;x<10;x++)
    for(int y=-10;y<10;y++)
        sm.SetPixel(v.x_+x,v.y_+y,c);
```
