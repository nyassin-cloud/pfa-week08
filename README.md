# pfa-week08
Midterm

## Sketch Bloom Pro — Live Studio v4

A drawing-to-polygons tool for Autodesk Maya with editable strokes, filled silhouettes, animal and object presets, and a purple/pink/blue interface.

### Run in Maya

1. Open Maya’s Script Editor and select a **Python** tab.
2. Copy the entire Python script below into the editor.
3. Run it to open **Sketch Bloom — Live Studio v4**.

### Using the tool

- Drag empty space to draw; click an existing drawing to select and edit its points.
- Choose **Tube**, **Curve**, or **Filled shape**. Filled shapes close and fill on mouse release.
- Finished drawings become editable Maya objects automatically.
- **Brush & tool polygons** controls samples, tube sides, subdivision, roundness, depth segments, and an approximate polygon target.
- **Clear Canvas** removes this Studio’s drawings; unrelated scene objects are preserved.
- The shape library includes basic shapes, nature, objects, and animal head/full-body silhouette variants.
- Use the window edges to resize; **F11** toggles fullscreen.

### Testing status

Python compilation, Qt interaction/resizing, and preset/fill geometry checks pass. Automatic scene-update logic has also been checked with a simulated Maya command scene. Native scene integration and embedded viewport interaction still require verification in Maya.

### Complete Python script

```python
"""Sketch Bloom: run this file in Maya's Python Script Editor."""
import math
# Simple Studio v3 — direct drawing and immediate click editing.
from contextlib import contextmanager
from maya import cmds, OpenMayaUI
from maya.api import OpenMaya as om
from maya.api import OpenMayaUI as omui
try:
    from PySide6 import QtCore, QtGui, QtWidgets
    from shiboken6 import wrapInstance
except ImportError:
    from PySide2 import QtCore, QtGui, QtWidgets
    from shiboken2 import wrapInstance


def cross(a, b, c):
    return (b[0]-a[0])*(c[1]-a[1]) - (b[1]-a[1])*(c[0]-a[0])

def simplify_path(points,tolerance,closed=False):
    """Remove input jitter without cutting across important outline corners."""
    clean=[]
    for p in points:
        if not clean or math.dist(clean[-1],p)>1e-8: clean.append(tuple(p))
    if closed and len(clean)>2 and math.dist(clean[0],clean[-1])<tolerance: clean.pop()
    def reduce_line(pts):
        if len(pts)<3: return pts
        a,b=pts[0],pts[-1]; dx,dy=b[0]-a[0],b[1]-a[1]; length=dx*dx+dy*dy
        distances=[]
        for p in pts[1:-1]:
            t=max(0,min(1,((p[0]-a[0])*dx+(p[1]-a[1])*dy)/max(length,1e-20)))
            distances.append(math.hypot(p[0]-a[0]-t*dx,p[1]-a[1]-t*dy))
        index=max(range(len(distances)),key=distances.__getitem__)+1
        if distances[index-1]<=tolerance: return [a,b]
        return reduce_line(pts[:index+1])[:-1]+reduce_line(pts[index:])
    if len(clean)<4: return clean
    if closed:
        split=max(range(1,len(clean)),key=lambda i:math.dist(clean[0],clean[i]))
        return reduce_line(clean[:split+1])[:-1]+reduce_line(clean[split:]+[clean[0]])[:-1]
    return reduce_line(clean)

def fill_contours(points):
    """Resolve a hand drawn outline with Qt's winding fill rule."""
    if len(points)<3: raise ValueError('Draw an outline with at least three points.')
    # Qt path operations are more stable at pixel-like coordinates.
    lo=[min(p[k] for p in points) for k in (0,1)]
    extent=max(max(p[k] for p in points)-lo[k] for k in (0,1))
    if extent<1e-9: raise ValueError('Draw an outline with some area.')
    if len(points)>2048: raise ValueError('This outline is too dense. Reduce its points before filling.')
    clean=simplify_path(points,extent*1e-7,True)
    try:
        triangulate(clean)
        return [clean]
    except ValueError:
        pass
    scale=1000/extent
    path=QtGui.QPainterPath(); path.setFillRule(QtCore.Qt.WindingFill)
    path.moveTo((points[0][0]-lo[0])*scale,(points[0][1]-lo[1])*scale)
    for p in points[1:]: path.lineTo((p[0]-lo[0])*scale,(p[1]-lo[1])*scale)
    path.closeSubpath(); result=[]
    for polygon in path.simplified().toSubpathPolygons():
        loop=[(p.x()/scale+lo[0],p.y()/scale+lo[1]) for p in polygon]
        loop=simplify_path(loop,.01/scale,True)
        if len(loop)>=3: result.append(loop)
    if not result: raise ValueError('This outline has no fillable area. Draw a wider shape.')
    return result


def triangulate(points):
    """Ear clipping, rejecting crossing/degenerate outlines."""
    n = len(points)
    if n < 3:
        raise ValueError('Draw a larger outline to see your mesh.')
    for i in range(n):
        a, b = points[i], points[(i+1)%n]
        for j in range(i+1, n):
            if j == i+1 or (i == 0 and j == n-1):
                continue
            c, d = points[j], points[(j+1)%n]
            if cross(a,b,c)*cross(a,b,d) <= 0 and cross(c,d,a)*cross(c,d,b) <= 0:
                if max(min(a[0],b[0]),min(c[0],d[0])) <= min(max(a[0],b[0]),max(c[0],d[0])) and max(min(a[1],b[1]),min(c[1],d[1])) <= min(max(a[1],b[1]),max(c[1],d[1])):
                    raise ValueError('Your outline crosses itself. Try a single simple loop.')
    area = sum(points[i][0]*points[(i+1)%n][1] - points[(i+1)%n][0]*points[i][1] for i in range(n))
    ids = list(range(n)) if area > 0 else list(reversed(range(n)))
    faces = []
    while len(ids) > 3:
        for k, b in enumerate(ids):
            a, c = ids[k-1], ids[(k+1)%len(ids)]
            if cross(points[a],points[b],points[c]) <= 1e-8:
                continue
            if any(all(v >= -1e-8 for v in (cross(points[a],points[b],points[t]), cross(points[b],points[c],points[t]), cross(points[c],points[a],points[t]))) for t in ids if t not in (a,b,c)):
                continue
            faces.append((a,b,c))
            ids.pop(k)
            break
        else:
            raise ValueError('This outline is too thin or tangled. Try drawing it again.')
    if abs(cross(*(points[i] for i in ids))) < 1e-8:
        raise ValueError('Draw a shape with a little more area.')
    return faces + [tuple(ids)]


@contextmanager
def transient():
    enabled = cmds.undoInfo(query=True, state=True)
    cmds.undoInfo(stateWithoutFlush=False)
    try:
        yield
    finally:
        cmds.undoInfo(stateWithoutFlush=enabled)



import os, time, copy, tempfile, json
from maya import OpenMaya as legacy_om

@contextmanager
def chunk(label):
    cmds.undoInfo(openChunk=True, chunkName=label)
    try:
        yield
    finally:
        cmds.undoInfo(closeChunk=True)

def sample(points, count, closed):
    if len(points)<2: raise ValueError('Draw a longer stroke.')
    lengths=[math.dist(points[i],points[(i+1)%len(points)]) for i in range(len(points) if closed else len(points)-1)]
    total=sum(lengths)
    if total<1e-8: raise ValueError('Draw a larger stroke.')
    result=[]; edge=0; start=0.0
    for i in range(count):
        target=total*i/(count if closed else count-1)
        while edge<len(lengths)-1 and start+lengths[edge]<target:
            start+=lengths[edge]; edge+=1
        a,b=points[edge],points[(edge+1)%len(points)]
        t=(target-start)/max(lengths[edge],1e-12)
        result.append(tuple(a[k]+t*(b[k]-a[k]) for k in range(len(a))))
    return result

def tube(points,radius,sides):
    pts=[om.MVector(*p) for p in points]; vertices=[]; faces=[]; normal=None
    if len(pts)<2: raise ValueError('Draw a longer stroke.')
    for i,center in enumerate(pts):
        tangent=pts[min(i+1,len(pts)-1)]-pts[max(0,i-1)]
        if tangent.length()<1e-10: tangent=om.MVector(0,1,0)
        tangent.normalize()
        if normal is not None: normal=normal-tangent*(normal*tangent)
        if normal is None or normal.length()<1e-6:
            axis=om.MVector(0,1,0) if abs(tangent.y)<.9 else om.MVector(1,0,0)
            normal=tangent^axis
        normal.normalize(); binormal=(tangent^normal).normal()
        for j in range(sides):
            a=j*math.pi*2/sides
            v=center+radius*(normal*math.cos(a)+binormal*math.sin(a))
            vertices.append((v.x,v.y,v.z))
    for i in range(len(pts)-1):
        for j in range(sides):
            k=(j+1)%sides
            faces.append([i*sides+j,i*sides+k,(i+1)*sides+k,(i+1)*sides+j])
    faces += [list(reversed(range(sides))),[(len(pts)-1)*sides+j for j in range(sides)]]
    return vertices,faces

def create_mesh(geometry,name):
    vertices,faces=geometry
    obj=om.MFnMesh().create([om.MPoint(*v) for v in vertices],[len(f) for f in faces],[i for f in faces for i in f])
    parent=obj if obj.hasFn(om.MFn.kTransform) else om.MFnDagNode(obj).parent(0)
    node=cmds.rename(om.MFnDagNode(parent).fullPathName(),name)
    cmds.sets(node,edit=True,forceElement='initialShadingGroup')
    return node

def selected_mesh():
    for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
        shapes=[item] if cmds.nodeType(item)=='mesh' else cmds.listRelatives(item,shapes=True,noIntermediate=True,fullPath=True,type='mesh') or []
        if shapes: return cmds.listRelatives(shapes[0],parent=True,fullPath=True)[0],shapes[0]
    raise ValueError('Select a polygon mesh in Maya first.')

def flower_icon():
    img=QtGui.QImage(32,32,QtGui.QImage.Format_ARGB32); img.fill(QtCore.Qt.transparent)
    p=QtGui.QPainter(img); p.setRenderHint(QtGui.QPainter.Antialiasing)
    p.setPen(QtGui.QPen(QtGui.QColor('#739AB8'),1)); p.setBrush(QtGui.QColor('#9CCDF4'))
    for i in range(5):
        a=i*math.pi*2/5-math.pi/2
        p.drawEllipse(QtCore.QPointF(16+7*math.cos(a),16+7*math.sin(a)),6,6)
    p.setBrush(QtGui.QColor('#EDA7D5')); p.drawEllipse(QtCore.QPointF(16,16),5,5); p.end()
    path=os.path.join(tempfile.gettempdir(),'bloom_pro_flower.png'); img.save(path)
    return path.replace('\\','/')


def add_cap(vertices,faces,ring,style):
    if style=='Open': return
    if style in ('N-gons','Quads • subdivided'):
        faces.append(list(ring)); return
    center=len(vertices)
    vertices.append(tuple(sum(vertices[i][k] for i in ring)/len(ring) for k in (0,1,2)))
    if style=='Quad fan • convex':
        if len(ring)%2: raise ValueError('Quad fans need an even boundary count.')
        for i in range(0,len(ring),2): faces.append([center,ring[i],ring[(i+1)%len(ring)],ring[(i+2)%len(ring)]])
    else:
        for i in range(len(ring)): faces.append([center,ring[i],ring[(i+1)%len(ring)]])

def split_quads(vertices,faces):
    result=list(vertices); edge_points={}; output=[]
    def midpoint(a,b):
        key=tuple(sorted((a,b)))
        if key not in edge_points:
            edge_points[key]=len(result)
            result.append(tuple((vertices[a][k]+vertices[b][k])/2 for k in (0,1,2)))
        return edge_points[key]
    for face in faces:
        center=len(result)
        result.append(tuple(sum(vertices[i][k] for i in face)/len(face) for k in (0,1,2)))
        for j,index in enumerate(face):
            output.append([index,midpoint(index,face[(j+1)%len(face)]),center,midpoint(face[j-1],index)])
    return result,output


def make_editable(node):
    shapes=cmds.listRelatives(node,shapes=True,noIntermediate=True,fullPath=True) or []
    for item in [node]+shapes:
        for attr,value in [('overrideEnabled',False),('overrideDisplayType',0),('template',False)]:
            plug=item+'.'+attr
            if cmds.objExists(plug) and not cmds.getAttr(plug,lock=True):
                try: cmds.setAttr(plug,value)
                except RuntimeError: pass

def filled_geometry(points,normal,depth,segments,style):
    if len(points)<3: raise ValueError('Draw a larger loop for a filled solid.')
    n=om.MVector(normal).normal(); axis=om.MVector(0,1,0) if abs(n.y)<.9 else om.MVector(1,0,0)
    u=(n^axis).normal(); v=(n^u).normal(); origin=om.MVector(*points[0])
    polygon=[]
    for point in points:
        relative=om.MVector(*point)-origin; polygon.append((relative*u,relative*v))
    vertices=[]; faces=[]
    # Filled shapes always have real triangulated caps, including concave shapes.
    # Tube cap settings such as Open or a convex fan cannot remove their fill.
    for loop in fill_contours(polygon):
        area=sum(loop[i][0]*loop[(i+1)%len(loop)][1]-loop[(i+1)%len(loop)][0]*loop[i][1] for i in range(len(loop)))
        if area<0: loop.reverse()
        triangles=triangulate(loop); count=len(loop); base=len(vertices)
        for layer in range(segments+1):
            z=depth*(layer/segments-.5)
            for x,y in loop:
                p=origin+u*x+v*y+n*z; vertices.append((p.x,p.y,p.z))
        faces.extend([[base+i for i in reversed(t)] for t in triangles])
        faces.extend([[base+segments*count+i for i in t] for t in triangles])
        for layer in range(segments):
            for i in range(count):
                j=(i+1)%count
                faces.append([base+layer*count+i,base+layer*count+j,base+(layer+1)*count+j,base+(layer+1)*count+i])
    return split_quads(vertices,faces) if style=='Quads • subdivided' else (vertices,faces)


def cup_geometry(radius,height,wall,base,sides,vertical_segments):
    if wall>=radius or base>=height: raise ValueError('Cup wall must be smaller than radius, and base smaller than height.')
    profile=[(0,0),(radius,0)]
    profile.extend([(radius,height*i/vertical_segments) for i in range(1,vertical_segments+1)])
    profile.append((radius-wall,height))
    profile.extend([(radius-wall,height-(height-base)*i/vertical_segments) for i in range(1,vertical_segments+1)])
    profile.append((0,base))
    vertices=[]; rings=[]; faces=[]
    for r,y in profile:
        if r<1e-10:
            rings.append([len(vertices)]); vertices.append((0,y,0))
        else:
            ring=[]
            for j in range(sides):
                a=j*math.pi*2/sides; ring.append(len(vertices)); vertices.append((r*math.cos(a),y,r*math.sin(a)))
            rings.append(ring)
    for a,b in zip(rings,rings[1:]):
        if len(a)==1 and len(b)==1: continue
        for j in range(sides):
            k=(j+1)%sides
            if len(a)==1: faces.append([a[0],b[j],b[k]])
            elif len(b)==1: faces.append([a[j],b[0],a[k]])
            else: faces.append([a[j],b[j],b[k],a[k]])
    return vertices,faces

PRESET_GROUPS = {
    'Basics': ['Circle','Oval','Triangle','Square','Rectangle','Rounded rectangle','Diamond','Pentagon','Hexagon','Octagon','Trapezoid','Cross','Arrow','Lightning bolt'],
    'Nature': ['Star','Heart','Flower','Daisy','Sunflower','Tulip','Leaf','Tree','Pine tree','Cloud','Raindrop','Moon','Sun','Mountain','Mushroom'],
    'Animals': ['Cat','Dog','Bunny','Bear','Fox','Fish','Butterfly','Bird','Whale','Turtle'],
    'Things': ['Shoe','Boot','Sneaker','House','Crown','Cup','Bottle','Vase','Bag','T-shirt','Dress','Rocket','Car','Bow','Balloon'],
}

def preset_outline(name):
    """Simple closed silhouettes, rather than inferred 3D animal models."""
    profiles = {
        'Triangle': [(0,-1),(1,1),(-1,1)],
        'Square': [(-1,-1),(1,-1),(1,1),(-1,1)],
        'Rectangle': [(-1,-.6),(1,-.6),(1,.6),(-1,.6)],
        'Diamond': [(0,-1),(.65,0),(0,1),(-.65,0)],
        'Trapezoid': [(-.6,-1),(.6,-1),(1,1),(-1,1)],
        'Cross': [(-.3,-1),(.3,-1),(.3,-.3),(1,-.3),(1,.3),(.3,.3),(.3,1),(-.3,1),(-.3,.3),(-1,.3),(-1,-.3),(-.3,-.3)],
        'Arrow': [(-1,-.3),(.25,-.3),(.25,-.85),(1,0),(.25,.85),(.25,.3),(-1,.3)],
        'Lightning bolt': [(.1,-1),(-.8,.2),(-.15,.2),(-.4,1),(.85,-.3),(.2,-.3)],
        'Cat': [(-.85,-1),(-.35,-.65),(.35,-.65),(.85,-1),(.8,-.15),(1,.35),(.7,.85),(0,1),(-.7,.85),(-1,.35),(-.8,-.15)],
        'Dog': [(-.4,-.7),(.4,-.7),(.65,-1),(1,-.75),(.95,.3),(.6,.5),(.5,.15),(.55,.7),(.3,1),(-.3,1),(-.55,.7),(-.5,.15),(-.6,.5),(-.95,.3),(-1,-.75),(-.65,-1)],
        'Bunny': [(-.7,-1),(-.35,-.95),(-.2,-.3),(.2,-.3),(.35,-.95),(.7,-1),(.7,-.5),(.5,0),(.85,.4),(.7,.85),(0,1),(-.7,.85),(-.85,.4),(-.5,0),(-.7,-.5)],
        'Bear': [(-.8,-.25),(-1,-.65),(-.9,-.95),(-.55,-1),(-.3,-.7),(.3,-.7),(.55,-1),(.9,-.95),(1,-.65),(.8,-.25),(.95,.4),(.65,.85),(0,1),(-.65,.85),(-.95,.4)],
        'Fox': [(-1,-1),(-.25,-.6),(.25,-.6),(1,-1),(.85,.25),(0,1),(-.85,.25)],
        'Fish': [(-1,-.7),(-.45,-.25),(0,-.6),(.6,-.5),(1,0),(.6,.5),(0,.6),(-.45,.25),(-1,.7),(-.8,0)],
        'Bird': [(-1,.5),(-.4,.25),(-.8,-.6),(-.15,-.2),(.3,-.75),(.65,-.7),(.8,-.35),(1,-.2),(.7,-.05),(.5,.55),(0,.8),(-.5,.65)],
        'Whale': [(-1,-.6),(-.55,-.2),(-.25,-.6),(.4,-.8),(.85,-.55),(1,0),(.85,.55),(.4,.75),(-.2,.65),(-.55,.2),(-1,.5),(-.8,0)],
        'Turtle': [(-.8,-.35),(-1,-.65),(-.75,-.8),(-.5,-.55),(-.2,-.75),(.3,-.7),(.55,-.5),(.8,-.65),(1,-.5),(.85,-.15),(1,.05),(.85,.3),(.55,.25),(.4,.65),(.65,.8),(.4,1),(.1,.75),(-.4,.65),(-.65,.85),(-.9,.65),(-.7,.35),(-.9,0)],
        'Shoe': [(-1,-.6),(-.55,-.6),(-.45,-.1),(-.05,.1),(.6,.25),(1,.55),(.95,.85),(-1,.85)],
        'Sneaker': [(-1,-.3),(-.6,-.55),(-.25,-.15),(.05,-.35),(.45,.15),(.9,.35),(1,.6),(.95,.85),(-1,.85)],
        'Boot': [(-.9,-1),(-.15,-1),(-.15,.2),(.55,.3),(.95,.55),(1,.85),(-.9,.85)],
        'House': [(-1,-.1),(0,-1),(1,-.1),(.75,-.1),(.75,1),(-.75,1),(-.75,-.1)],
        'Crown': [(-1,-.8),(-.5,-.15),(0,-1),(.5,-.15),(1,-.8),(.75,.8),(-.75,.8)],
        'Cup': [(-.8,-.8),(.45,-.8),(.45,-.55),(.9,-.55),(1,-.2),(.95,.3),(.45,.5),(.3,.85),(-.55,.85)],
        'Bottle': [(-.3,-1),(.3,-1),(.3,-.55),(.65,-.25),(.65,.85),(.45,1),(-.45,1),(-.65,.85),(-.65,-.25),(-.3,-.55)],
        'Vase': [(-.5,-1),(.5,-1),(.35,-.4),(.8,.3),(.7,.75),(.4,1),(-.4,1),(-.7,.75),(-.8,.3),(-.35,-.4)],
        'Bag': [(-.8,-.4),(-.5,-.4),(-.5,-.8),(-.3,-1),(.3,-1),(.5,-.8),(.5,-.4),(.8,-.4),(1,1),(-1,1)],
        'T-shirt': [(-.45,-.8),(-.2,-.6),(.2,-.6),(.45,-.8),(1,-.4),(.7,.1),(.45,-.05),(.45,1),(-.45,1),(-.45,-.05),(-.7,.1),(-1,-.4)],
        'Dress': [(-.45,-1),(-.15,-.8),(.15,-.8),(.45,-1),(.6,-.55),(.3,-.2),(.25,.1),(.9,1),(-.9,1),(-.25,.1),(-.3,-.2),(-.6,-.55)],
        'Rocket': [(0,-1),(.4,-.5),(.4,.2),(.85,.7),(.85,1),(.25,.7),(-.25,.7),(-.85,1),(-.85,.7),(-.4,.2),(-.4,-.5)],
        'Car': [(-1,0),(-.65,-.1),(-.35,-.65),(.35,-.65),(.65,-.1),(.95,.05),(1,.55),(.65,.55),(.55,.85),(.3,.85),(.2,.55),(-.35,.55),(-.45,.85),(-.7,.85),(-.8,.55),(-1,.55)],
        'Bow': [(-1,-.7),(-.2,-.15),(.2,-.15),(1,-.7),(.8,0),(1,.7),(.2,.15),(-.2,.15),(-1,.7),(-.8,0)],
        'Tulip': [(-.8,-1),(-.35,-.6),(0,-1),(.35,-.6),(.8,-1),(.65,-.2),(.15,.15),(.15,.5),(.65,.2),(.45,.7),(.15,.8),(.15,1),(-.15,1),(-.15,.85),(-.45,.75),(-.65,.25),(-.15,.55),(-.15,.15),(-.65,-.2)],
        'Tree': [(-.2,1),(-.2,.35),(-.75,.4),(-1,0),(-.8,-.5),(-.35,-.65),(0,-1),(.4,-.8),(.55,-.5),(.9,-.35),(1,.1),(.7,.4),(.2,.35),(.2,1)],
        'Pine tree': [(0,-1),(.55,-.35),(.3,-.35),(.8,.25),(.45,.25),(1,.8),(.2,.8),(.2,1),(-.2,1),(-.2,.8),(-1,.8),(-.45,.25),(-.8,.25),(-.3,-.35),(-.55,-.35)],
        'Mountain': [(-1,1),(-.3,-1),(.05,-.1),(.4,-.65),(1,1)],
        'Mushroom': [(-1,.1),(-.9,-.5),(-.5,-.9),(0,-1),(.5,-.9),(.9,-.5),(1,.1),(.3,.1),(.45,1),(-.45,1),(-.3,.1)],
    }
    if name in profiles: raw=profiles[name]
    elif name in ('Pentagon','Hexagon','Octagon'):
        count={'Pentagon':5,'Hexagon':6,'Octagon':8}[name]
        raw=[(math.cos(2*math.pi*i/count-math.pi/2),math.sin(2*math.pi*i/count-math.pi/2)) for i in range(count)]
    elif name=='Rounded rectangle':
        raw=[]
        for cx,cy,start in [(.65,-.45,-90),(.65,.45,0),(-.65,.45,90),(-.65,-.45,180)]:
            for i in range(13):
                a=math.radians(start+i*90/12); raw.append((cx+.25*math.cos(a),cy+.25*math.sin(a)))
    elif name=='Cloud':
        # One outline assembled from an upper scalloped edge and a flat base.
        raw=[(-.95,.5),(-1,.15),(-.9,-.2),(-.6,-.3),(-.5,-.65),(-.2,-.85),(.15,-.8),(.35,-.5),(.65,-.55),(.9,-.3),(1,.15),(.9,.5)]
    elif name=='Moon':
        raw=[(math.cos(a),math.sin(a)) for a in [math.radians(65+230*i/64) for i in range(65)]]
        bottom,top=raw[-1],raw[0]
        raw.extend([(bottom[0]-.85*math.sin(math.pi*t),bottom[1]+(top[1]-bottom[1])*t) for t in [i/64 for i in range(1,64)]])
    else:
        raw=[]; count=160
        for i in range(count):
            t=2*math.pi*i/count
            if name=='Heart': x=16*math.sin(t)**3/17; y=-(13*math.cos(t)-5*math.cos(2*t)-2*math.cos(3*t)-math.cos(4*t))/17
            elif name=='Star':
                # Use exact corners below rather than a wavy star.
                continue
            elif name in ('Flower','Daisy','Sunflower','Sun','Butterfly'):
                petals={'Flower':5,'Daisy':8,'Sunflower':12,'Sun':16,'Butterfly':4}[name]
                r=.72+.25*math.cos(petals*t); x=r*math.cos(t); y=r*math.sin(t)
                if name=='Butterfly': x*=1.15; y*=.85
            elif name=='Leaf': x=.55*math.sin(t); y=math.cos(t)*(1+.15*math.sin(t))
            elif name=='Raindrop': x=.7*math.sin(t)*(1-.45*math.cos(t)); y=-math.cos(t)
            elif name=='Balloon': x=.7*math.cos(t); y=.85*math.sin(t)
            else: x=math.cos(t); y=math.sin(t)*(.65 if name=='Oval' else 1)
            raw.append((x,y))
        if name=='Star': raw=[((1 if i%2==0 else .45)*math.cos(i*math.pi/5-math.pi/2),(1 if i%2==0 else .45)*math.sin(i*math.pi/5-math.pi/2)) for i in range(10)]
    lo=[min(p[k] for p in raw) for k in (0,1)]; hi=[max(p[k] for p in raw) for k in (0,1)]
    scale=.8/max(hi[0]-lo[0],hi[1]-lo[1])
    return [(.5+(x-(lo[0]+hi[0])/2)*scale,.5+(y-(lo[1]+hi[1])/2)*scale) for x,y in raw]

ANIMAL_BODIES = {
    'Cat': [(-.9,.85),(-.7,.45),(-.65,-.2),(-.9,-.65),(-.85,-1),(-.5,-.8),(-.15,-.9),(.1,-1),(.25,-.6),(.15,-.3),(.4,.1),(.55,.55),(.85,.5),(1,.1),(1,.65),(.75,.9),(.35,1),(.15,.85),(-.15,.85),(-.25,1),(-.7,1)],
    'Dog': [(-1,-.7),(-.65,-.8),(-.45,-.55),(-.2,-.55),(-.15,-.3),(-.35,-.2),(-.35,.1),(.25,.1),(.6,-.05),(.85,-.45),(1,-.4),(.85,.05),(.6,.3),(.65,.9),(.4,.9),(.3,.45),(-.1,.45),(-.2,.9),(-.45,.9),(-.45,.3),(-.65,.15),(-.65,-.15),(-.85,-.2),(-1,-.4)],
    'Bunny': [(-.8,-1),(-.5,-.95),(-.25,-.3),(-.05,-.35),(.15,-1),(.45,-1),(.4,-.55),(.2,-.1),(.45,.1),(.55,.4),(.85,.65),(.9,.9),(.6,1),(-.4,1),(-.6,.85),(-.9,.8),(-1,.6),(-.8,.4),(-.65,.5),(-.5,.2),(-.65,-.1)],
    'Bear': [(-.9,-.5),(-1,-.8),(-.8,-1),(-.55,-.9),(-.25,-.9),(-.05,-1),(.2,-.9),(.2,-.6),(.5,-.3),(.6,.15),(.85,.45),(.65,.65),(.45,.4),(.45,.85),(.25,1),(0,.85),(-.25,1),(-.55,.85),(-.5,.4),(-.75,.65),(-.95,.4),(-.75,.1),(-.65,-.3)],
    'Fox': [(-1,-1),(-.55,-.75),(-.2,-1),(-.1,-.5),(-.3,-.2),(-.15,.2),(.15,.6),(.5,.5),(.7,.05),(.65,-.45),(1,-.2),(1,.5),(.75,.9),(.2,1),(-.2,.85),(-.4,.95),(-.65,.95),(-.55,.55),(-.75,.1),(-.85,-.4)],
    'Fish': [(-1,-.7),(-.45,-.2),(-.1,-.5),(.15,-.9),(.4,-.5),(.75,-.3),(1,0),(.75,.35),(.4,.5),(.1,.9),(-.05,.55),(-.45,.2),(-1,.7),(-.8,0)],
    'Bird': [(-.9,-.4),(-.65,-.8),(-.25,-.8),(-.05,-.55),(.2,-.4),(.45,-.05),(.85,.2),(1,.05),(.9,.5),(.6,.7),(.15,.7),(.15,1),(0,1),(-.05,.7),(-.25,.65),(-.35,.95),(-.5,.95),(-.4,.5),(-.6,.25),(-.65,-.2),(-1,-.25)],
    'Whale': [(-1,-.6),(-.55,-.2),(-.25,-.6),(.4,-.8),(.85,-.55),(1,0),(.85,.55),(.4,.75),(.1,.65),(-.1,1),(-.3,.9),(-.2,.55),(-.55,.2),(-1,.5),(-.8,0)],
    'Turtle': [(-1,-.2),(-.75,-.4),(-.8,-.7),(-.55,-.65),(-.4,-.8),(.05,-.9),(.4,-.7),(.6,-.35),(.75,-.45),(1,-.3),(1,0),(.8,.15),(.55,.1),(.4,.4),(.55,.65),(.3,.7),(.15,.45),(-.3,.45),(-.45,.75),(-.65,.65),(-.55,.3),(-.8,.1)],
    'Butterfly': [(0,-.8),(.15,-.9),(.2,-.65),(.55,-1),(.95,-.85),(1,-.4),(.65,.1),(.85,.5),(.6,.9),(.25,.7),(.1,.3),(0,1),(-.1,.3),(-.25,.7),(-.6,.9),(-.85,.5),(-.65,.1),(-1,-.4),(-.95,-.85),(-.55,-1),(-.2,-.65),(-.15,-.9)],
}

def animal_outline(name,full_body=False,realistic=False):
    if full_body:
        raw=list(ANIMAL_BODIES[name])
        if realistic:
            # Longer torsos and narrower limbs; smooth silhouettes rather than cartoon corners.
            raw=[(x*1.25,y*.9) for x,y in raw]
    else:
        if realistic and name in ('Dog','Fox'):
            raw=[(-1,-.7),(-.6,-1),(-.25,-.7),(.2,-.65),(.4,-.3),(1,-.05),(.95,.25),(.45,.4),(.15,.85),(-.4,1),(-.75,.55),(-.9,0)]
        elif realistic and name=='Cat':
            raw=[(-.85,-1),(-.35,-.65),(.3,-.65),(.8,-1),(.75,-.15),(.95,.2),(.6,.55),(.25,.65),(0,.85),(-.3,.65),(-.65,.55),(-.95,.2),(-.75,-.15)]
        elif name in ('Cat','Dog','Bunny','Bear','Fox'):
            raw=preset_outline(name)
        elif name=='Bird': raw=[(-.7,-.8),(-.1,-1),(.4,-.75),(.5,-.25),(1,0),(.45,.2),(.2,.8),(-.5,1),(-.85,.3)]
        elif name in ('Fish','Whale','Turtle'):
            raw=[(math.cos(i*math.pi/32),math.sin(i*math.pi/32)*(.7 if name!='Turtle' else .9)) for i in range(64)]
        else:
            raw=[(-.8,-1),(-.4,-.9),(-.2,-.35),(.2,-.35),(.4,-.9),(.8,-1),(.55,-.55),(.45,.1),(.65,.5),(.3,1),(-.3,1),(-.65,.5),(-.45,.1),(-.55,-.55)]
    if realistic:
        for unused in range(2):
            smooth=[]
            for a,b in zip(raw,raw[1:]+raw[:1]):
                smooth.extend([(a[0]*.8+b[0]*.2,a[1]*.8+b[1]*.2),(a[0]*.2+b[0]*.8,a[1]*.2+b[1]*.8)])
            raw=smooth
    lo=[min(p[k] for p in raw) for k in (0,1)]; hi=[max(p[k] for p in raw) for k in (0,1)]
    scale=.8/max(hi[0]-lo[0],hi[1]-lo[1])
    return [(.5+(x-(lo[0]+hi[0])/2)*scale,.5+(y-(lo[1]+hi[1])/2)*scale) for x,y in raw]

class SketchCanvas(QtWidgets.QWidget):
    changed=QtCore.Signal()
    selection_changed=QtCore.Signal()
    def __init__(self):
        super().__init__(); self.strokes=[]; self.selected=-1; self.history=[]; self.future=[]
        self.mode='Sketch'; self.active_point=-1; self.zoom=1.0; self.pan=QtCore.QPointF(0,0); self.editing=False; self.moving_stroke=False
        self.panning=False; self.drawing=False; self.dragging=False; self.last_mouse=None
        self.setMinimumSize(160,160); self.setFocusPolicy(QtCore.Qt.StrongFocus); self.setCursor(QtCore.Qt.CrossCursor)
    def remember(self):
        self.history.append((copy.deepcopy(self.strokes),self.selected)); self.history=self.history[-60:]; self.future=[]
    def finish(self):
        self.changed.emit(); self.selection_changed.emit(); self.update()
    def undo(self):
        if self.history:
            self.future.append((copy.deepcopy(self.strokes),self.selected)); self.strokes,self.selected=self.history.pop(); self.finish()
    def redo(self):
        if self.future:
            self.history.append((copy.deepcopy(self.strokes),self.selected)); self.strokes,self.selected=self.future.pop(); self.finish()
    def current(self):
        return self.strokes[self.selected] if 0<=self.selected<len(self.strokes) else None
    def scale(self): return max(1,min(self.width(),self.height())-32)*self.zoom
    def screen(self,p):
        s=self.scale(); return QtCore.QPointF(self.width()/2+self.pan.x()+(p[0]-.5)*s,self.height()/2+self.pan.y()+(p[1]-.5)*s)
    def world(self,p):
        s=self.scale(); return ((p.x()-self.width()/2-self.pan.x())/s+.5,(p.y()-self.height()/2-self.pan.y())/s+.5)
    def event_pos(self,e): return e.position() if hasattr(e,'position') else e.localPos()
    def fit(self):
        points=[p for s in self.strokes for p in s['points']]; self.zoom=1; self.pan=QtCore.QPointF(0,0)
        if points:
            lo=[min(p[k] for p in points) for k in (0,1)]; hi=[max(p[k] for p in points) for k in (0,1)]
            self.zoom=min(20,.82/max(hi[0]-lo[0],hi[1]-lo[1],.05)); size=self.scale()
            self.pan=QtCore.QPointF((.5-(lo[0]+hi[0])/2)*size,(.5-(lo[1]+hi[1])/2)*size)
        self.update()
    def wheelEvent(self,e):
        pos=self.event_pos(e); point=self.world(pos)
        self.zoom=max(.1,min(30,self.zoom*(1.15 if e.angleDelta().y()>0 else 1/1.15)))
        self.pan+=pos-self.screen(point); self.update()
    def mousePressEvent(self,e):
        self.setFocus(); pos=self.event_pos(e)
        if e.button()==QtCore.Qt.MiddleButton:
            self.panning=True; self.last_mouse=pos; return
        if e.button()!=QtCore.Qt.LeftButton: return
        # Every drawing tool supports clicking an existing drawing immediately.
        # Hold Shift to begin a new stroke over an existing one.
        force_draw=bool(e.modifiers() & QtCore.Qt.ShiftModifier)
        s=self.current()
        if s and self.editing and not force_draw:
            nearest=min(((math.dist((self.screen(p).x(),self.screen(p).y()),(pos.x(),pos.y())),i) for i,p in enumerate(s['points'])),default=(999,-1))
            if nearest[0]<10:
                self.remember(); self.active_point=nearest[1]; self.dragging=True; self.finish(); return
        if not force_draw:
            for si in reversed(range(len(self.strokes))):
                stroke=self.strokes[si]; pts=stroke['points']
                if not pts: continue
                path=QtGui.QPainterPath(); path.moveTo(self.screen(pts[0]))
                for point in pts[1:]: path.lineTo(self.screen(point))
                if stroke['closed']: path.closeSubpath()
                stroker=QtGui.QPainterPathStroker(); stroker.setWidth(16)
                if stroker.createStroke(path).contains(pos) or (stroke['closed'] and path.contains(pos)):
                    self.remember(); self.selected=si; self.active_point=-1; self.editing=True
                    self.moving_stroke=True; self.last_mouse=pos; self.finish(); return
        self.remember(); auto_fill=self.mode=='Continuous fill'
        self.strokes.append({'uid':str(time.time_ns()),'points':[self.world(pos)],'closed':auto_fill,'build':getattr(self,'new_build',0 if auto_fill else 1),'mesh_settings':copy.deepcopy(getattr(self,'default_mesh_settings',{}))})
        self.selected=len(self.strokes)-1; self.drawing=True; self.active_point=-1; self.editing=False
        self.finish()
    def mouseMoveEvent(self,e):
        pos=self.event_pos(e)
        if self.panning:
            self.pan+=pos-self.last_mouse; self.last_mouse=pos; self.update(); return
        s=self.current()
        if s and self.moving_stroke:
            a,b=self.world(self.last_mouse),self.world(pos); dx,dy=b[0]-a[0],b[1]-a[1]
            s['points']=[(p[0]+dx,p[1]+dy) for p in s['points']]; self.last_mouse=pos; self.changed.emit(); self.update()
        elif s and self.drawing:
            point=self.world(pos)
            if math.dist(point,s['points'][-1])*self.scale()>2: s['points'].append(point)
            self.changed.emit(); self.update()
        elif s and self.dragging and self.active_point>=0:
            s['points'][self.active_point]=self.world(pos); self.changed.emit(); self.update()
    def mouseReleaseEvent(self,e):
        s=self.current()
        if self.drawing and s:
            if len(s['points'])<2: self.strokes.pop(self.selected); self.selected=-1
            elif len(s['points'])>48: s['points']=simplify_path(s['points'],.002,s['closed'])
        self.panning=False; self.drawing=False; self.dragging=False; self.moving_stroke=False; self.finish()
    def mouseDoubleClickEvent(self,e):
        if not self.current(): return
        s=self.current(); pos=self.event_pos(e); pts=s['points']; nearest=None
        for i in range(len(pts) if s['closed'] else len(pts)-1):
            a,b=self.screen(pts[i]),self.screen(pts[(i+1)%len(pts)])
            dx,dy=b.x()-a.x(),b.y()-a.y()
            t=max(0,min(1,((pos.x()-a.x())*dx+(pos.y()-a.y())*dy)/max(dx*dx+dy*dy,1e-10)))
            q=QtCore.QPointF(a.x()+t*dx,a.y()+t*dy); d=math.hypot(q.x()-pos.x(),q.y()-pos.y())
            if nearest is None or d<nearest[0]: nearest=(d,i,self.world(q))
        if nearest and nearest[0]<20:
            self.remember(); pts.insert(nearest[1]+1,nearest[2]); self.finish()
    def keyPressEvent(self,e):
        if e.key()==QtCore.Qt.Key_Delete and self.current():
            self.remember(); s=self.current()
            if self.active_point>=0 and len(s['points'])>(3 if s['closed'] else 2):
                s['points'].pop(self.active_point); self.active_point=-1
            else:
                self.strokes.pop(self.selected); self.selected=min(self.selected,len(self.strokes)-1)
            self.finish()
        else: super().keyPressEvent(e)
    def paintEvent(self,e):
        p=QtGui.QPainter(self); p.setRenderHint(QtGui.QPainter.Antialiasing); p.fillRect(self.rect(),QtGui.QColor('#FAF5FF'))
        p.setPen(QtGui.QColor('#D7C9EE')); step=max(12,self.scale()/20)
        x=(self.width()/2+self.pan.x())%step
        while x<self.width():
            y=(self.height()/2+self.pan.y())%step
            while y<self.height(): p.drawPoint(QtCore.QPointF(x,y)); y+=step
            x+=step
        for si,s in enumerate(self.strokes):
            if not s['points']: continue
            path=QtGui.QPainterPath(); path.setFillRule(QtCore.Qt.WindingFill); path.moveTo(self.screen(s['points'][0]))
            for point in s['points'][1:]: path.lineTo(self.screen(point))
            if s['closed']:
                path.closeSubpath(); p.fillPath(path,QtGui.QColor(185,163,238,200))
            p.setPen(QtGui.QPen(QtGui.QColor('#CB4E9C' if si==self.selected else '#666AC5'),3.2 if si==self.selected else 2.3)); p.drawPath(path)
            if si==self.selected and self.editing:
                p.setBrush(QtGui.QColor('#F7B5DF'))
                for pi,point in enumerate(s['points']):
                    p.drawEllipse(self.screen(point),6 if pi==self.active_point else 4,6 if pi==self.active_point else 4)
        if not self.strokes:
            p.setPen(QtGui.QColor('#76638F')); p.drawText(self.rect(),QtCore.Qt.AlignCenter,'Drag here to draw. Click a drawing to edit it.\n\nMiddle-drag to pan • Wheel to zoom')
        p.end()


class SceneHotkeys(QtCore.QObject):
    def __init__(self,view):
        super().__init__(view); self.view=view
    def eventFilter(self,watched,event):
        owner=self.view.studio
        if owner is None: return False
        native=self.view.native_widget
        if native is None or not isinstance(watched,QtWidgets.QWidget): return False
        if watched is not native and not native.isAncestorOf(watched): return False
        kind=event.type()
        if kind in (QtCore.QEvent.MouseButtonPress,QtCore.QEvent.MouseMove,QtCore.QEvent.MouseButtonRelease):
            return self.view.pointer_event(watched,event)
        if kind not in (QtCore.QEvent.KeyPress,QtCore.QEvent.ShortcutOverride): return False
        focus=QtWidgets.QApplication.focusWidget()
        if isinstance(focus,(QtWidgets.QLineEdit,QtWidgets.QAbstractSpinBox,QtWidgets.QComboBox,QtWidgets.QTextEdit,QtWidgets.QPlainTextEdit)): return False
        if event.modifiers() & QtCore.Qt.AltModifier: return False
        key=event.key(); qt=QtCore.Qt
        handled={qt.Key_Q,qt.Key_W,qt.Key_E,qt.Key_R,qt.Key_F,qt.Key_A,qt.Key_Z,qt.Key_Y,qt.Key_Escape,qt.Key_F8,qt.Key_F9,qt.Key_F10,qt.Key_F11,qt.Key_1,qt.Key_2,qt.Key_3,qt.Key_4,qt.Key_5,qt.Key_6,qt.Key_7}
        if key not in handled or (key==qt.Key_Y and not event.modifiers() & qt.ControlModifier): return False
        if event.modifiers() & qt.ControlModifier and key not in (qt.Key_Z,qt.Key_Y): return False
        if kind==QtCore.QEvent.ShortcutOverride: event.accept(); return True
        try:
            if key in (qt.Key_W,qt.Key_E,qt.Key_R):
                owner.scene_transform({qt.Key_W:'moveSuperContext',qt.Key_E:'RotateSuperContext',qt.Key_R:'scaleSuperContext'}[key])
            elif key in (qt.Key_Q,qt.Key_Escape):
                owner.live_tool.stop(); cmds.setToolTo('selectSuperContext'); owner.native_edit=False
                if key==qt.Key_Escape and owner.isFullScreen(): owner.normal_window()
            elif key==qt.Key_F11: owner.toggle_fullscreen()
            elif key==qt.Key_F: self.view.frame()
            elif key==qt.Key_A: self.view.frame_all()
            elif key==qt.Key_Z:
                owner.live_tool.stop()
                if event.modifiers() & qt.ShiftModifier: cmds.redo()
                else: cmds.undo()
            elif key==qt.Key_Y and event.modifiers() & qt.ControlModifier: owner.live_tool.stop(); cmds.redo()
            elif key==qt.Key_F8:
                self.view.component('vertex' if cmds.selectMode(query=True,object=True) else 'object')
            elif key in (qt.Key_F9,qt.Key_F10):
                self.view.component({qt.Key_F9:'vertex',qt.Key_F10:'edge'}[key])
            elif key in (qt.Key_1,qt.Key_2,qt.Key_3):
                for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
                    shapes=[item] if cmds.nodeType(item)=='mesh' else cmds.listRelatives(item,shapes=True,noIntermediate=True,fullPath=True,type='mesh') or []
                    for shape in shapes:
                        if cmds.objExists(shape+'.displaySmoothMesh'): cmds.setAttr(shape+'.displaySmoothMesh',{qt.Key_1:0,qt.Key_2:1,qt.Key_3:2}[key])
            elif key in (qt.Key_4,qt.Key_5,qt.Key_6,qt.Key_7):
                panel=self.view.active_panel()
                if panel:
                    if key==qt.Key_4: cmds.modelEditor(panel,edit=True,displayAppearance='wireframe')
                    else: cmds.modelEditor(panel,edit=True,displayAppearance='smoothShaded',displayTextures=key==qt.Key_6,displayLights='all' if key==qt.Key_7 else 'default')
        except Exception as error: owner.status.setText('Viewport shortcut: '+str(error))
        event.accept(); return True

class SceneViews(QtWidgets.QWidget):
    def __init__(self,cameras):
        super().__init__(); self.cameras=cameras; self.panels=[]; self.host=None; self.ready=False; self.studio=None; self.native_widget=None; self.hotkey_filter=SceneHotkeys(self); QtWidgets.QApplication.instance().installEventFilter(self.hotkey_filter); self.setFocusPolicy(QtCore.Qt.StrongFocus)
        self.box=QtWidgets.QVBoxLayout(self); self.box.setContentsMargins(0,0,0,0)
        hint=QtWidgets.QLabel('Drag empty space to draw • Click an object to select • Alt + mouse navigates • Q returns from point editing to drawing'); hint.setWordWrap(True); self.box.addWidget(hint)
        self.message=QtWidgets.QLabel('Opening Maya viewport…'); self.message.setWordWrap(True); self.box.addWidget(self.message)
        self.content=QtWidgets.QSplitter(QtCore.Qt.Horizontal); self.viewport=QtWidgets.QWidget(); self.viewport_box=QtWidgets.QVBoxLayout(self.viewport); self.viewport_box.setContentsMargins(0,0,0,0); self.viewport.setMinimumSize(80,80); self.viewport.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Ignored); self.content.addWidget(self.viewport); self.box.addWidget(self.content,1)
        self.input_active=False
    def showEvent(self,event):
        super().showEvent(event)
        if not self.ready: self.ready=True; QtCore.QTimer.singleShot(0,self.build)
    def build(self):
        if self.native_widget is not None: return
        try:
            self.host=cmds.window(title='Bloom embedded views')
            pane=cmds.paneLayout(parent=self.host,configuration='quad' if len(self.cameras)>1 else 'single')
            for camera in self.cameras:
                panel=cmds.modelPanel(parent=pane,camera=camera,menuBarVisible=False); self.panels.append(panel)
                cmds.modelEditor(panel,edit=True,displayAppearance='smoothShaded',displayLights='default',wireframeOnShaded=True,grid=True)
            cmds.showWindow(self.host)
            pointer=OpenMayaUI.MQtUtil.findLayout(pane)
            if not pointer: raise RuntimeError('Maya did not return the viewport layout.')
            widget=wrapInstance(int(pointer),QtWidgets.QWidget); widget.setParent(self.viewport); widget.setSizePolicy(QtWidgets.QSizePolicy.Expanding,QtWidgets.QSizePolicy.Expanding); widget.setFocusPolicy(QtCore.Qt.StrongFocus); self.viewport_box.addWidget(widget,1); widget.show(); self.native_widget=widget; self.message.hide()
            host_pointer=OpenMayaUI.MQtUtil.findWindow(self.host)
            if host_pointer: wrapInstance(int(host_pointer),QtWidgets.QWidget).hide()
            widget.setMinimumSize(0,0)
            QtCore.QTimer.singleShot(100,self.focus_scene)
        except Exception as error:
            self.message.setText('Viewport could not be embedded: '+str(error)+'\nUse Maya’s main viewport. Live Draw also works there.')
    def ensure_ready(self):
        if self.native_widget is None: self.build()
        if self.native_widget is not None:
            self.native_widget.show(); self.native_widget.update(); self.focus_scene()
    def pointer_event(self,watched,event):
        owner=self.studio
        if owner is None or not hasattr(owner,'live_output'): return False
        kind=event.type(); qt=QtCore.Qt
        if event.modifiers() & qt.AltModifier: return False
        if kind==QtCore.QEvent.MouseButtonPress and event.button()!=qt.LeftButton: return False
        if kind==QtCore.QEvent.MouseMove and not self.input_active: return False
        if kind==QtCore.QEvent.MouseButtonRelease and not self.input_active: return False
        if owner.native_edit and not self.input_active: return False
        try:
            found=None
            for panel in self.panels:
                view=omui.M3dView.getM3dViewFromModelPanel(panel)
                widget=wrapInstance(int(view.widget()),QtWidgets.QWidget)
                if watched is widget or widget.isAncestorOf(watched): found=(panel,view,widget); break
            if found is None: return False
            panel,view,widget=found
            global_pos=event.globalPosition().toPoint() if hasattr(event,'globalPosition') else event.globalPos()
            p=widget.mapFromGlobal(global_pos)
            screen=(p.x()*view.portWidth()/max(1,widget.width()),(widget.height()-1-p.y())*view.portHeight()/max(1,widget.height()))
            if kind==QtCore.QEvent.MouseButtonPress:
                cmds.setFocus(panel); cmds.modelEditor(panel,edit=True,activeView=True)
                if not event.modifiers() & qt.ShiftModifier:
                    cmds.selectMode(object=True); cmds.selectType(polymesh=True,nurbsCurve=True)
                    om.MGlobal.selectFromScreen(int(screen[0]),int(screen[1]),om.MGlobal.kReplaceList,om.MGlobal.kSurfaceSelectMethod)
                    picked=cmds.ls(selection=True,objectsOnly=True,long=True) or []
                    target_ids=set(owner.paintable_shapes) if owner.live_surface.isChecked() else set()
                    selectable=[]
                    for node in picked:
                        shapes=[node] if cmds.nodeType(node)=='mesh' else cmds.listRelatives(node,shapes=True,noIntermediate=True,fullPath=True) or []
                        ids={uid for shape in shapes for uid in (cmds.ls(shape,uuid=True) or [])}
                        if not ids.intersection(target_ids): selectable.append(node)
                    if selectable:
                        cmds.select(selectable,replace=True); owner.status.setText('Selected '+selectable[0].split('|')[-1]+'. Drag with Move or choose Points to reshape.'); return True
                owner.live_tool.direct_input=True; owner.live_tool.begin(view,screen); self.input_active=True
            elif kind==QtCore.QEvent.MouseMove: owner.live_tool.move(screen)
            elif kind==QtCore.QEvent.MouseButtonRelease:
                owner.live_tool.move(screen); owner.live_tool.release(); self.input_active=False
            event.accept(); return True
        except Exception as error:
            self.input_active=False; owner.live_tool.finalize(); owner.status.setText('Viewport: '+str(error)); return True
    def active_panel(self):
        for value in [cmds.getPanel(underPointer=True),cmds.getPanel(withFocus=True)]:
            if value in self.panels: return value
        return self.panels[0] if self.panels else None
    def focus_scene(self):
        panel=self.active_panel()
        if panel and cmds.modelPanel(panel,exists=True):
            try:
                cmds.setFocus(panel); cmds.modelEditor(panel,edit=True,activeView=True)
            except RuntimeError: pass
    def frame_all(self):
        for panel in self.panels:
            if cmds.modelPanel(panel,exists=True): cmds.viewFit(cmds.modelPanel(panel,query=True,camera=True),allObjects=True)
    def frame(self):
        for panel in self.panels:
            if cmds.modelPanel(panel,exists=True):
                camera=cmds.modelPanel(panel,query=True,camera=True)
                cmds.viewFit(camera,allObjects=not bool(cmds.ls(selection=True)))
    def component(self,kind):
        if kind=='object': cmds.selectMode(object=True)
        else:
            cmds.selectMode(component=True); cmds.selectType(allComponents=False)
            cmds.selectType(**{{'vertex':'polymeshVertex','edge':'polymeshEdge','face':'polymeshFace'}[kind]:True})
        cmds.setToolTo('selectSuperContext')
    def cleanup(self):
        QtWidgets.QApplication.instance().removeEventFilter(self.hotkey_filter)
        for panel in self.panels:
            try:
                if cmds.modelPanel(panel,exists=True): cmds.deleteUI(panel,panel=True)
            except Exception: pass
        try:
            if self.host and cmds.window(self.host,exists=True): cmds.deleteUI(self.host,window=True)
        except Exception: pass

class LiveStrokeTool:
    def __init__(self,owner):
        self.owner=owner; self.context='SketchBloomLiveDrawContext'; self.points=[]; self.preview=None; self.target=None; self.last_update=0; self.direct_input=False
    def activate(self):
        if cmds.currentCtx()==self.context: cmds.setToolTo('selectSuperContext')
        if self.owner.live_output.currentIndex()==2 and self.owner.live_surface.isChecked():
            raise ValueError('Filled solids use the camera plane. Turn surface projection off, or choose tubes/curves.')
        if self.owner.live_surface.isChecked():
            paths=self.owner.paintable_paths()
            if not paths:
                self.owner.set_paintable(); paths=self.owner.paintable_paths()
            if not paths: raise ValueError('Select a character and click Make Selected Character Paintable first.')
            self.target=paths
        else: self.target=None
        if cmds.draggerContext(self.context,exists=True): cmds.deleteUI(self.context,toolContext=True)
        cmds.draggerContext(self.context,pressCommand=self.press,dragCommand=self.drag,releaseCommand=self.release,finalize=self.finalize,space='screen',cursor='crossHair',undoMode='step')
        cmds.setToolTo(self.context)
        self.owner.status.setText('Live Draw active. Draw in a Maya viewport. Stop to select or edit; orbit between strokes.')
    def remove_preview(self):
        if self.preview and cmds.objExists(self.preview): cmds.delete(self.preview)
        self.preview=None
    def stop(self):
        if cmds.currentCtx()==self.context: cmds.setToolTo('selectSuperContext')
        self.finalize()
    def finalize(self,*args):
        with transient(): self.remove_preview()
        self.points=[]
    def ray(self,screen):
        if isinstance(self.view,omui.M3dView):
            source,direction=om.MPoint(),om.MVector()
            self.view.viewToWorld(int(screen[0]),int(screen[1]),source,direction)
            return source,direction.normal()
        source,direction=legacy_om.MPoint(),legacy_om.MVector()
        self.view.viewToWorld(int(screen[0]),int(screen[1]),source,direction)
        return om.MPoint(source.x,source.y,source.z),om.MVector(direction.x,direction.y,direction.z).normal()
    def projected(self,screen):
        source,ray=self.ray(screen)
        if self.target is not None:
            nearest=None
            for path in self.target:
                fn=om.MFnMesh(path)
                hit=fn.closestIntersection(om.MFloatPoint(source.x,source.y,source.z),om.MFloatVector(ray.x,ray.y,ray.z),om.MSpace.kWorld,1000000.0,False,accelParams=fn.autoUniformGridParams())
                if hit and (nearest is None or hit[1]<nearest[0]): nearest=(hit[1],hit,fn)
            if nearest is None: return None
            hit,fn=nearest[1],nearest[2]; p=hit[0]; normal=fn.getPolygonNormal(hit[2],om.MSpace.kWorld).normal(); offset=self.owner.surface_offset.value()
            return (p.x+normal.x*offset,p.y+normal.y*offset,p.z+normal.z*offset)
        denom=ray*self.normal
        if abs(denom)<1e-8: return None
        distance=((self.plane_point-source)*self.normal)/denom
        if distance<0: return None
        point=source+ray*distance
        return (point.x,point.y,point.z)
    def press(self,*args):
        self.direct_input=False
        self.begin(OpenMayaUI.M3dView.active3dView(),cmds.draggerContext(self.context,query=True,anchorPoint=True))
    def begin(self,view,screen):
        try:
            self.settings=copy.deepcopy(self.owner.current_shape_settings())
            self.view=view
            self.target=self.owner.paintable_paths() if self.owner.live_surface.isChecked() else None
            if self.owner.live_surface.isChecked() and not self.target: raise ValueError('Select a surface and click Use selected surface first.')
            source,self.normal=self.ray((self.view.portWidth()/2,self.view.portHeight()/2))
            self.plane_point=source+self.normal*self.owner.live_distance.value()
            if self.owner.live_plane.currentIndex()==1: self.plane_point=om.MPoint(0,0,0)
            elif self.owner.live_plane.currentIndex()==2:
                selected=cmds.ls(selection=True,objectsOnly=True,long=True) or []
                if not selected: raise ValueError('Select an object to position the drawing plane.')
                bounds=cmds.exactWorldBoundingBox(selected); self.plane_point=om.MPoint((bounds[0]+bounds[3])/2,(bounds[1]+bounds[4])/2,(bounds[2]+bounds[5])/2)
            with transient(): self.remove_preview()
            self.points=[]; point=self.projected(screen)
            if point: self.points=[point]
            self.last_update=0
        except Exception as error:
            self.points=[]; self.owner.status.setText('Live Draw: '+str(error))
    def drag(self,*args):
        self.move(cmds.draggerContext(self.context,query=True,dragPoint=True))
    def move(self,screen):
        try:
            point=self.projected(screen)
            if point is None:
                if self.points: self.release()
                return
            if not self.points or math.dist(point,self.points[-1])>self.owner.live_spacing.value(): self.points.append(point)
            if time.monotonic()-self.last_update>.08:
                self.update_preview(); self.last_update=time.monotonic()
        except Exception as error: self.owner.status.setText('Live Draw: '+str(error))
    def update_preview(self,draft=True):
        minimum=3 if self.owner.live_output.currentIndex()==2 else 2
        if len(self.points)<minimum: return
        new_node=None
        with transient():
            selected=cmds.ls(selection=True,long=True) or []
            try:
                if self.owner.live_output.currentIndex()==1 or (draft and self.owner.live_output.currentIndex()==2):
                    count=min(getattr(self,'settings',{}).get('samples',self.owner.resolution.value()),64) if draft else getattr(self,'settings',{}).get('samples',self.owner.resolution.value())
                    points=sample(self.points,count,False)
                    if self.owner.live_output.currentIndex()==2 and len(points)>2: points.append(points[0])
                    new_node=cmds.curve(degree=1,point=points,name='bloomLivePreview#')
                else:
                    geometry=self.owner.live_geometry(self.points,self.normal,draft=draft)
                    new_node=create_mesh(geometry,'bloomLivePreview#')
                self.remove_preview(); self.preview=new_node
            except Exception:
                if new_node and cmds.objExists(new_node): cmds.delete(new_node)
                raise
            finally:
                valid=[x for x in selected if cmds.objExists(x)]
                cmds.select(valid,replace=True) if valid else cmds.select(clear=True)
        cmds.refresh()
    def release(self,*args):
        try:
            minimum=3 if self.owner.live_output.currentIndex()==2 else 2
            if len(self.points)>=minimum:
                self.update_preview(draft=False)
                if self.preview:
                    node=None; finish_note=''
                    with chunk('Bloom live stroke'):
                        try:
                            node=cmds.duplicate(self.preview,returnRootsOnly=True,name='bloomStroke#')[0]
                            shapes=cmds.listRelatives(node,shapes=True,noIntermediate=True,type='mesh') or []
                            if shapes:
                                levels=getattr(self,'settings',{}).get('subdivision',self.owner.subdivision.value())
                                faces=cmds.polyEvaluate(node,face=True)
                                if faces*4**levels*2>200000: raise ValueError('Live result is too dense. Reduce subdivision, samples, sides or target.')
                                if levels: cmds.polySmooth(node,divisions=levels,continuity=getattr(self,'settings',{}).get('roundness',self.owner.rounding.value()),constructionHistory=False,keepBorder=True)
                                cmds.polySoftEdge(node,angle=60,constructionHistory=False)
                            make_editable(node); cmds.selectMode(object=True); cmds.select(node,replace=True)
                            cmds.addAttr(node,longName='bloomStrokeData',dataType='string')
                            cmds.setAttr(node+'.bloomStrokeData',json.dumps({'points':self.points,'normal':[self.normal.x,self.normal.y,self.normal.z],'output':self.owner.live_output.currentIndex(),'settings':getattr(self,'settings',{})}),type='string')
                        except Exception:
                            if node and cmds.objExists(node): cmds.delete(node)
                            raise
                    self.owner.register_object(node)
                    self.owner.viewport_strokes.append(cmds.ls(node,uuid=True)[0])
                    actual=' • {:,} faces'.format(cmds.polyEvaluate(node,face=True)) if shapes else ' • editable curve'
                    self.owner.status.setText('Created '+node+actual+'. '+finish_note+' Click an older object in Select mode to edit it.')
        except Exception as error: self.owner.status.setText('Drawing: '+str(error))
        finally:
            with transient(): self.remove_preview()
            self.points=[]
            if not self.direct_input and not self.owner.keep_drawing.isChecked():
                QtCore.QTimer.singleShot(0,lambda:self.owner.select_edit(keep_message=True))

class BloomPro(QtWidgets.QDialog):
    def __init__(self,parent):
        super().__init__(parent); self.setObjectName('SketchBloomProWindow'); self.setWindowTitle('Sketch Bloom • Live Studio v4')
        self.setWindowFlags(QtCore.Qt.Window | QtCore.Qt.WindowMinMaxButtonsHint | QtCore.Qt.WindowCloseButtonHint)
        self.setMinimumSize(420,320); self.setMaximumSize(16777215,16777215); self.resize(1100,780); self.setSizeGripEnabled(True); self.native_edit=False
        self.window_before_fullscreen=None
        self.preview=None; self.previews={}; self.preview_inputs={}; self.preview_busy=False; self.created_nodes=[]; self.scene_pickers=[]; self.scene_stats_labels=[]; self.paintable_shapes=[]; self.surface_labels=[]; self.views=[]; self.retopo_source=None; self.dirty=False; self.live_tool=LiveStrokeTool(self)
        self.scene_nodes={}; self.live_signatures={}; self.hidden_live={}; self.viewport_strokes=[]; self.last_edit=0
        self.setStyleSheet('''
            QDialog {background:qlineargradient(x1:0,y1:0,x2:1,y2:1,stop:0 #F3FAFF,stop:1 #FFF9E9);}
            QWidget {color:#243D58;font-family:Segoe UI;font-size:13px;}
            QLabel {background:transparent;color:#243D58;} QLabel#title {font-size:30px;font-weight:700;color:#426F91;}
            QLabel#status {background:#E4F1FB;color:#355774;border:1px solid #BED8EB;border-radius:10px;padding:10px;}
            QGroupBox {background:#E5F3FD;color:#243D58;border:1px solid #C2DDED;border-radius:13px;margin-top:26px;padding:17px 12px 12px;}
            QGroupBox::title {subcontrol-origin:margin;background:#FFF1BE;color:#365976;padding:4px 10px;border-radius:7px;left:12px;}
            QLineEdit,QSpinBox,QDoubleSpinBox,QComboBox {background:#FFFBEB;color:#284660;border:1px solid #D7CD9F;border-radius:8px;padding:7px;min-height:20px;selection-background-color:#B7DBFA;selection-color:#243D58;}
            QLineEdit:focus,QSpinBox:focus,QDoubleSpinBox:focus,QComboBox:focus {border:2px solid #7DAFD1;}
            QComboBox QAbstractItemView {background:#FFF7D6;color:#243D58;selection-background-color:#B7DBFA;selection-color:#243D58;}
            QPushButton {background:#D9EDFC;color:#284660;border:1px solid #B5D5EC;border-radius:10px;padding:10px;}
            QPushButton:hover {background:#C3E2F8;border-color:#8EBBDB;} QPushButton:pressed {background:#B1D5EE;}
            QPushButton#make {background:qlineargradient(x1:0,y1:0,x2:1,y2:0,stop:0 #FFE59B,stop:1 #FFF0BC);color:#50451F;border:1px solid #D9C27C;font-size:15px;font-weight:700;padding:14px;border-radius:12px;}
            QPushButton#make:hover {background:#FFDD7A;}
            QCheckBox {background:transparent;color:#243D58;spacing:8px;min-height:29px;}
            QCheckBox::indicator {width:26px;height:26px;background:#FFF7D6;border:2px solid #739AB8;border-radius:5px;}
            QCheckBox::indicator:checked {background:#FFF7D6;border:2px solid #739AB8;border-radius:5px;image:url("FLOWER");}
            QScrollArea {border:none;background:#F0F8FF;} QScrollArea > QWidget > QWidget {background:#F0F8FF;}
            QTabBar::tab {background:#DEEEFA;color:#243D58;border:1px solid #BDD7E8;border-bottom:none;border-top-left-radius:10px;border-top-right-radius:10px;margin-right:4px;padding:11px 13px;}
            QTabBar::tab:selected {background:#FFF1BE;color:#355774;font-weight:700;} QTabBar::tab:hover {background:#CEE6F7;}
            QTabWidget::pane {border:1px solid #BDD7E8;border-radius:9px;background:#F4FAFE;} QSplitter::handle {background:#D2E7F5;width:6px;}
            QToolTip {background:#FFFAE5;color:#284660;border:1px solid #D5C68D;padding:7px;}
        '''.replace('FLOWER',flower_icon()))
        root=QtWidgets.QVBoxLayout(self); root.setSizeConstraint(QtWidgets.QLayout.SetNoConstraint); root.setContentsMargins(10,8,10,8); root.setSpacing(6)
        heading=QtWidgets.QLabel('Sketch Bloom  ✿  3D Studio'); heading.setObjectName('title'); root.addWidget(heading)
        subtitle=QtWidgets.QLabel('A little sketch. A little shape. A whole world to make.'); root.addWidget(subtitle)
        self.tabs=QtWidgets.QTabWidget(); self.tabs.setUsesScrollButtons(True); self.tabs.setMinimumSize(0,0); self.tabs.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Ignored); root.addWidget(self.tabs,1)
        self.status=QtWidgets.QLabel('Drag empty space to draw. Click a drawing to edit.'); self.status.setWordWrap(True); self.status.setMaximumHeight(55); self.status.setObjectName('status'); root.addWidget(self.status)
        self.build_sketch()
        for title,cameras in [('Front',['front']),('Side',['side']),('Perspective',['persp']),('All Views',['persp','front','side','top'])]:
            view=SceneViews(cameras); self.views.append(view); self.tabs.addTab(view,title)
        self.build_live(); self.build_retopo(); self.build_assembly(); self.build_help(); self.add_view_draw_buttons()
        self.build_primary_tools(root)
        self.apply_bloom_theme()
        self.install_brush_shortcuts()
        self.timer=QtCore.QTimer(self); self.timer.setInterval(150); self.timer.timeout.connect(self.safe_update_preview); self.timer.start()
        self.selection_timer=QtCore.QTimer(self); self.selection_timer.setInterval(600); self.selection_timer.timeout.connect(self.update_selection_stats); self.selection_timer.start()
        shortcut_class=getattr(QtGui,'QShortcut',None) or QtWidgets.QShortcut
        # The viewport event filter uses the same fullscreen action for F11.
        self.fullscreen_shortcuts=[]
        for key in ('F11','Ctrl+Alt+F'):
            shortcut=shortcut_class(QtGui.QKeySequence(key),self); shortcut.setContext(QtCore.Qt.WidgetWithChildrenShortcut); shortcut.activated.connect(self.toggle_fullscreen); self.fullscreen_shortcuts.append(shortcut)
    def normal_window(self):
        self.showNormal()
        if self.window_before_fullscreen is not None: self.setGeometry(self.window_before_fullscreen)
        self.window_before_fullscreen=None
    def toggle_fullscreen(self):
        if self.isFullScreen(): self.normal_window()
        else:
            self.window_before_fullscreen=self.normalGeometry(); self.showFullScreen()
    def maximize_window(self):
        self.normal_window(); self.showMaximized()
    def reject(self):
        if self.isFullScreen(): self.normal_window()
        else:
            self.live_tool.stop(); self.native_edit=False; self.canvas.editing=False; self.canvas.update(); cmds.setToolTo('selectSuperContext')
    def button(self,label,callback):
        w=QtWidgets.QPushButton(label); w.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Preferred); w.setToolTip(label); w.clicked.connect(lambda checked=False:self.run_action(callback)); return w
    def run_action(self,callback):
        try: callback()
        except Exception as error: self.status.setText(str(error)); cmds.warning('Sketch Bloom: '+str(error))
    def build_primary_tools(self,root):
        box=QtWidgets.QGroupBox('Tools'); grid=QtWidgets.QGridLayout(box); grid.setContentsMargins(8,18,8,8)
        grid.addWidget(QtWidgets.QLabel('Draw as'),0,0); grid.addWidget(self.live_output,0,1)
        grid.addWidget(QtWidgets.QLabel('Size'),1,0); size=self.mirror_control(self.brush_size); size.setToolTip('Tube diameter'); grid.addWidget(size,1,1)
        grid.addWidget(QtWidgets.QLabel('Thickness'),2,0); grid.addWidget(self.depth,2,1)
        grid.addWidget(self.button('✿ Fill selected object',self.fill_selected),3,0,1,2)
        grid.addWidget(self.button('Move',lambda:self.scene_transform('moveSuperContext')),4,0)
        grid.addWidget(self.button('Edit points',self.edit_active_points),4,1)
        grid.addWidget(self.button('Use selected surface',self.set_paintable),5,0,1,2)
        grid.addWidget(self.live_surface,6,0,1,2); grid.addWidget(self.button('Clear surface',self.clear_paintable),7,0,1,2)
        grid.setColumnStretch(0,1); grid.setColumnStretch(1,1)
        # Each view has its own controls widget, all synchronized with the same sources.
        self.sketch_controls_layout.insertWidget(0,box)
        for view in self.views:
            if view is self.sketch_perspective: continue
            scroll=QtWidgets.QScrollArea(); scroll.setWidgetResizable(True); scroll.setMinimumSize(0,0)
            panel=QtWidgets.QWidget(); side=QtWidgets.QVBoxLayout(panel); scroll.setWidget(panel)
            self.group(side,'Tools',[('Draw as',self.mirror_control(self.live_output)),('Size',self.mirror_control(self.brush_size)),('Thickness',self.mirror_control(self.depth)),('',self.button('✿ Fill selected object',self.fill_selected)),('',self.button('Move',lambda:self.scene_transform('moveSuperContext'))),('',self.button('Edit points',self.edit_active_points)),('',self.button('Use selected surface',self.set_paintable)),('',self.mirror_control(self.live_surface)),('',self.button('Clear surface',self.clear_paintable)),('',self.button('Clear Canvas ✿',self.clear_canvas))])
            self.group(side,'Mesh finish',[('Subdivision',self.mirror_control(self.subdivision)),('Roundness',self.mirror_control(self.rounding)),('Target faces ≈',self.mirror_control(self.face_target)),('',self.mirror_control(self.use_face_target))])
            stats=QtWidgets.QLabel('Click a mesh to see its actual polygon count.'); stats.setWordWrap(True); self.scene_stats_labels.append(stats); side.addWidget(stats); side.addStretch()
            view.content.addWidget(scroll); view.content.setSizes([760,300])
    def active_view(self):
        widget=self.tabs.currentWidget()
        if widget is self.tabs.widget(0) and self.sketch_stack.currentIndex()==1: return self.sketch_perspective
        return widget if isinstance(widget,SceneViews) else None
    def focus_active_view(self):
        view=self.active_view()
        if view: view.ensure_ready(); view.focus_scene()
        return view
    def fit_active_view(self):
        view=self.focus_active_view()
        if view: view.frame()
        else: self.canvas.fit()
    def keep_selected_drawing(self):
        self.dirty=True; self.update_preview()

    def edit_active_points(self):
        if self.active_view(): self.edit_scene_points()
        else:
            self.canvas.editing=True; self.canvas.update(); self.canvas.setFocus(); self.status.setText('Click a drawing, then drag any visible point. Double-click an edge to insert a point.')
    def surface_mode_changed(self,enabled):
        if enabled and not self.paintable_shapes: self.set_paintable()
        if enabled and self.live_output.currentIndex()==2: self.live_output.setCurrentIndex(0)
        if enabled and not self.paintable_shapes:
            self.live_surface.blockSignals(True); self.live_surface.setChecked(False); self.live_surface.blockSignals(False)
    def apply_bloom_theme(self):
        self.setStyleSheet('''
            QDialog {background:qlineargradient(x1:0,y1:0,x2:1,y2:1,stop:0 #EDE9FF,stop:.5 #F5E5FA,stop:1 #E4F0FF);}
            QWidget {font-family:Segoe UI;font-size:13px;color:#34234F;}
            QLabel {background:transparent;}
            QLabel#title {font-size:28px;font-weight:700;color:#75539A;}
            QLabel#status {background:#E5E9FF;border:1px solid #C3B4E8;border-radius:10px;padding:8px;}
            QGroupBox {background:qlineargradient(x1:0,y1:0,x2:1,y2:1,stop:0 #EEE5FF,stop:1 #E3EDFF);border:1px solid #C7B4E5;border-radius:13px;margin-top:22px;padding:12px 10px 10px;}
            QGroupBox::title {subcontrol-origin:margin;background:#F6DDF1;padding:4px 10px;border-radius:7px;left:12px;}
            QLineEdit,QSpinBox,QDoubleSpinBox,QComboBox {background:#FCF7FF;color:#34234F;border:1px solid #BFA9DA;border-radius:8px;padding:6px;min-height:20px;selection-background-color:#D3D9FF;}
            QComboBox QAbstractItemView {background:#FCF7FF;color:#34234F;selection-background-color:#DCD1FF;}
            QPushButton {background:qlineargradient(x1:0,y1:0,x2:1,y2:1,stop:0 #E5DAFF,stop:1 #D9E9FF);color:#34234F;border:1px solid #B5A1D5;border-radius:10px;padding:8px;}
            QPushButton:hover {background:#F0C7E5;} QPushButton:pressed {background:#C5CFFF;} QPushButton:checked {background:#EDC4E6;}
            QPushButton#make {background:qlineargradient(x1:0,y1:0,x2:1,y2:0,stop:0 #D5C1FA,stop:.5 #F0C4E5,stop:1 #CADFFF);font-size:15px;font-weight:700;padding:12px;border-radius:12px;}
            QCheckBox {background:transparent;spacing:5px;}
            QCheckBox::indicator {width:26px;height:26px;border:2px solid #9B7FBF;border-radius:5px;background:#FBF5FF;}
            QCheckBox::indicator:checked {background:#F8E6F5;border:2px solid #9B7FBF;image:url("FLOWER");}
            QScrollArea {border:none;background:#F4EFFF;} QScrollArea > QWidget > QWidget {background:#F4EFFF;}
            QTabBar::tab {background:#DED9F8;color:#34234F;border:1px solid #BEB0DA;padding:9px 12px;margin-right:4px;border-top-left-radius:10px;border-top-right-radius:10px;}
            QTabBar::tab:selected {background:#F2D3ED;font-weight:700;} QTabBar::tab:hover {background:#DDD0FA;} QTabWidget::pane {border:1px solid #C5B7E0;border-radius:9px;}
            QSplitter::handle {background:#CBBCEC;width:7px;height:7px;}
            QToolTip {background:#F8E9F6;color:#34234F;border:1px solid #BDA7D8;padding:5px;}
        '''.replace('FLOWER',flower_icon()))
    def combo(self,items):
        w=QtWidgets.QComboBox(); w.addItems(items); return w
    def integer(self,minimum,maximum,value):
        w=QtWidgets.QSpinBox(); w.setRange(minimum,maximum); w.setValue(value); w.setKeyboardTracking(False); return w
    def number(self,minimum,maximum,value):
        w=QtWidgets.QDoubleSpinBox(); w.setDecimals(3); w.setRange(minimum,maximum); w.setValue(value); w.setKeyboardTracking(False); return w
    def group(self,layout,title,rows):
        box=QtWidgets.QGroupBox(title); form=QtWidgets.QFormLayout(box); form.setRowWrapPolicy(QtWidgets.QFormLayout.WrapLongRows)
        for label,w in rows: form.addRow(label,w) if label else form.addRow(w)
        layout.addWidget(box)
    def build_sketch(self):
        page=QtWidgets.QWidget(); layout=QtWidgets.QVBoxLayout(page); layout.setContentsMargins(4,4,4,4)
        split=QtWidgets.QSplitter(QtCore.Qt.Horizontal); layout.addWidget(split,1)
        left=QtWidgets.QWidget(); left.setMinimumSize(0,0); left.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Ignored); ll=QtWidgets.QVBoxLayout(left); ll.setContentsMargins(0,0,0,0); ll.setSizeConstraint(QtWidgets.QLayout.SetNoConstraint)
        self.canvas=SketchCanvas(); self.sketch_camera=self.combo(['2D drawing','3D drawing'])
        ll.addWidget(self.sketch_camera)
        self.sketch_stack=QtWidgets.QStackedWidget(); self.sketch_stack.setMinimumSize(0,0)
        self.sketch_stack.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Ignored)
        self.sketch_stack.addWidget(self.canvas); self.sketch_perspective=SceneViews(['persp']); self.sketch_stack.addWidget(self.sketch_perspective); self.views.append(self.sketch_perspective)
        self.sketch_camera.currentIndexChanged.connect(self.change_sketch_camera); ll.addWidget(self.sketch_stack,1)
        hint=QtWidgets.QLabel('Drag empty space to draw. Click a drawing to edit it. Drag a point to reshape; drag the outline to move it. Shift-drag draws over an existing shape.'); hint.setWordWrap(True); ll.addWidget(hint)
        actions=QtWidgets.QHBoxLayout()
        for label,callback in [('Undo',self.sketch_undo),('Redo',self.sketch_redo),('Fit',self.fit_active_view)]: actions.addWidget(self.button(label,callback))
        ll.addLayout(actions)
        clear=self.button('Clear Canvas  ✿',self.clear_canvas); clear.setObjectName('make'); ll.addWidget(clear)
        split.addWidget(left)
        scroll=QtWidgets.QScrollArea(); scroll.setWidgetResizable(True); scroll.setMinimumSize(0,0); scroll.setSizePolicy(QtWidgets.QSizePolicy.Ignored,QtWidgets.QSizePolicy.Ignored)
        panel=QtWidgets.QWidget(); controls=QtWidgets.QVBoxLayout(panel); self.sketch_controls_layout=controls; scroll.setWidget(panel); split.addWidget(scroll); split.setSizes([780,280])
        # Internal controls remain available to the shared tool logic, without duplicate UI.
        self.mode=self.combo(['Sketch','Continuous fill']); self.stroke_picker=self.combo([])
        self.closed=QtWidgets.QCheckBox('Closed outline'); self.anchors=self.integer(4,160,24)
        self.output=self.combo(['Filled shape','Tube','Curve']); self.output.setCurrentIndex(1)
        self.plane=self.combo(['XY • front','YZ • side','XZ • top'])
        self.width_units=self.number(.1,1000,10); self.depth=self.number(.01,1000,.4)
        self.resolution=self.integer(8,1024,96); self.depth_segments=self.integer(1,128,1)
        self.radius=self.number(.001,100,.12); self.brush_size=self.number(.002,200,.24)
        self.brush_size.valueChanged.connect(lambda value:self.radius.setValue(value/2)); self.radius.valueChanged.connect(lambda value:self.brush_size.setValue(value*2))
        self.tube_sides=self.integer(3,64,12); self.caps=self.combo(['Triangles','Quads • subdivided','N-gons','Open'])
        self.face_target=self.integer(100,1000000,5000); self.use_face_target=QtWidgets.QCheckBox('Retopologize new meshes')
        self.subdivision=self.integer(0,4,0); self.rounding=self.number(0,1,0); self.live_preview=QtWidgets.QCheckBox('Live drawing'); self.live_preview.setChecked(True)
        self.live_preview.setToolTip('Drawing and edits update Maya automatically. Preview stays lightweight while dragging; subdivision applies on release.')
        self.mode.currentTextChanged.connect(self.change_mode); self.stroke_picker.currentIndexChanged.connect(self.choose_stroke); self.closed.toggled.connect(self.close_stroke)
        self.canvas.selection_changed.connect(self.sync_strokes); self.canvas.changed.connect(self.mark_dirty)
        self.preset_category=self.combo(list(PRESET_GROUPS)); self.preset_picker=self.combo(PRESET_GROUPS['Basics'])
        self.animal_part=self.combo(['Head','Full body']); self.animal_style=self.combo(['Stylized','Natural / realistic silhouette'])
        self.preset_category.currentTextChanged.connect(self.change_preset_category)
        self.group(controls,'Shape library',[('Category',self.preset_category),('Shape',self.preset_picker),('Animal part',self.animal_part),('Animal style',self.animal_style),('',self.button('Add shape',lambda:self.preset(self.preset_picker.currentText())))])
        self.animal_part.setEnabled(False); self.animal_style.setEnabled(False)
        self.shape_subdivision=self.subdivision; self.shape_roundness=self.rounding
        self.shape_layers=self.depth_segments; self.shape_target=self.face_target
        self.shape_use_target=self.use_face_target; self.shape_use_target.setText('Aim for polygon count')
        self.shape_counts=QtWidgets.QLabel('Add or click a shape to see its polygon count.'); self.shape_counts.setWordWrap(True)
        self.group(controls,'Brush & tool polygons',[('Path samples',self.resolution),('Tube sides',self.tube_sides),('Subdivision levels',self.shape_subdivision),('Roundness 0–1',self.shape_roundness),('Depth segments',self.shape_layers),('Target faces ≈',self.shape_target),('',self.shape_use_target),('',self.shape_counts),('',self.button('Subdivide selected Maya copy',self.subdivide_shape_copy))])
        for widget in [self.shape_subdivision,self.shape_roundness,self.shape_layers,self.shape_target,self.resolution,self.tube_sides,self.radius,self.depth,self.width_units]: widget.valueChanged.connect(self.shape_settings_changed)
        self.caps.currentIndexChanged.connect(self.shape_settings_changed); self.plane.currentIndexChanged.connect(self.shape_settings_changed)
        self.shape_use_target.toggled.connect(self.shape_settings_changed)
        self.shape_subdivision.setToolTip('Applies to the active brush and selected drawing. Preview is lightweight during a drag; subdivision applies automatically on release.')
        self.shape_roundness.setToolTip('0 subdivides while preserving the silhouette; 1 smooths and rounds it.')
        self.shape_target.setToolTip('Adjusts sampling or depth segments to approach this final polygon count. The result is approximate; no slow automatic retopology.')
        self.advanced_toggle=QtWidgets.QPushButton('More mesh settings ▸'); self.advanced_toggle.setCheckable(True); controls.addWidget(self.advanced_toggle)
        self.advanced_panel=QtWidgets.QWidget(); advanced=QtWidgets.QVBoxLayout(self.advanced_panel); advanced.setContentsMargins(0,0,0,0)
        self.group(advanced,'Mesh settings',[('Plane',self.plane),('Canvas units',self.width_units),('Tube caps',self.caps)])
        controls.addWidget(self.advanced_panel); self.advanced_panel.hide(); self.advanced_toggle.toggled.connect(self.advanced_panel.setVisible)
        controls.addStretch(); self.stats=QtWidgets.QLabel('Ready to draw'); self.stats.setWordWrap(True); layout.addWidget(self.stats)
        for w in [self.plane,self.width_units,self.depth,self.resolution,self.depth_segments,self.radius,self.tube_sides,self.caps,self.live_preview]:
            if isinstance(w,QtWidgets.QComboBox): w.currentIndexChanged.connect(self.mark_dirty)
            elif isinstance(w,QtWidgets.QCheckBox): w.toggled.connect(self.mark_dirty)
            else: w.valueChanged.connect(self.mark_dirty)
        self.tabs.addTab(page,'Draw')
        self.canvas.default_mesh_settings=self.current_shape_settings()

    def clear_sketch(self):
        self.canvas.remember(); self.canvas.strokes=[]; self.canvas.selected=-1; self.canvas.active_point=-1; self.canvas.finish()
    def change_mode(self,value): self.canvas.mode=value; self.canvas.update()
    def sync_strokes(self):
        self.stroke_picker.blockSignals(True); self.stroke_picker.clear()
        self.stroke_picker.addItems(['Stroke '+str(i+1)+(' • filled' if s['closed'] else ' • open') for i,s in enumerate(self.canvas.strokes)])
        self.stroke_picker.setCurrentIndex(self.canvas.selected); self.stroke_picker.blockSignals(False)
        stroke=self.canvas.current()
        self.closed.blockSignals(True); self.closed.setChecked(bool(stroke and stroke['closed'])); self.closed.blockSignals(False)
        if stroke and hasattr(self,'shape_subdivision'):
            settings=stroke.get('mesh_settings',{})
            self.syncing_settings=True
            try:
                for widget,key in [(self.subdivision,'subdivision'),(self.rounding,'roundness'),(self.depth_segments,'layers'),(self.face_target,'target'),(self.resolution,'samples'),(self.tube_sides,'sides'),(self.radius,'radius'),(self.depth,'depth'),(self.width_units,'units')]:
                    if key in settings: widget.setValue(settings[key])
                if 'caps' in settings: self.caps.setCurrentText(settings['caps'])
                if 'plane' in settings: self.plane.setCurrentIndex(settings['plane'])
                if 'use_target' in settings: self.use_face_target.setChecked(settings['use_target'])
            finally: self.syncing_settings=False
        if hasattr(self,'shape_subdivision'): self.canvas.default_mesh_settings=self.current_shape_settings()
        self.update_shape_counts()

    def current_shape_settings(self):
        return {'subdivision':self.subdivision.value(),'roundness':self.rounding.value(),'layers':self.depth_segments.value(),'target':self.face_target.value(),'use_target':self.use_face_target.isChecked(),'samples':self.resolution.value(),'sides':self.tube_sides.value(),'radius':self.radius.value(),'depth':self.depth.value(),'caps':self.caps.currentText(),'plane':self.plane.currentIndex(),'units':self.width_units.value()}
    def shape_settings_changed(self,*args):
        if getattr(self,'syncing_settings',False): return
        settings=self.current_shape_settings(); self.canvas.default_mesh_settings=copy.deepcopy(settings)
        if self.active_view():
            for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
                if cmds.nodeType(item) in ('mesh','nurbsCurve'): item=cmds.listRelatives(item,parent=True,fullPath=True)[0]
                if cmds.objExists(item+'.bloomStrokeData'):
                    uid=cmds.ls(item,uuid=True)[0]; key='viewport_'+uid
                    self.scene_nodes[key]=uid
                    data=json.loads(cmds.getAttr(item+'.bloomStrokeData'))
                    self.pending_viewport_update=(key,copy.deepcopy(settings),data)
                    break
        else:
            stroke=self.canvas.current()
            if stroke:
                self.canvas.remember(); stroke['mesh_settings']=copy.deepcopy(settings)
        self.mark_dirty(); self.update_shape_counts()

    def update_shape_counts(self):
        if not hasattr(self,'shape_counts'): return
        stroke=self.canvas.current()
        if not stroke: self.shape_counts.setText('Add or click a shape to see its polygon count.'); return
        if stroke.get('build')==2: self.shape_counts.setText('Curve: no polygon faces.'); return
        try:
            geometry=self.geometry(stroke=stroke); faces=geometry[1]; settings=stroke.get('mesh_settings',{})
            levels=settings.get('subdivision',self.subdivision.value())
            estimate=len(faces) if not levels else sum(len(face) for face in faces)*4**(levels-1)
            text='Base: {:,} faces\nAfter subdivision: ≈ {:,} faces'.format(len(faces),estimate)
            if settings.get('use_target',self.use_face_target.isChecked()): text+='\nRetopology target: ≈ {:,} faces'.format(settings.get('target',self.face_target.value()))
            if levels: text+='\nSubdivision updates automatically on release.'
            if estimate>2000000: text+='\nReduce subdivision: exceeds the density limit.'
            self.shape_counts.setText(text)
        except Exception as error: self.shape_counts.setText('Finish the outline to see its count. '+str(error))
    def subdivide_shape_copy(self):
        if self.shape_subdivision.value()<1: self.status.setText('Set subdivision to 1 or higher first.'); return
        self.retopo_source=None
        self.capture_source()
        if not self.retopo_source: return
        self.refine_levels.setValue(self.shape_subdivision.value()); self.refine_round.setValue(self.shape_roundness.value()); self.refine_copy()
    def choose_stroke(self,index):
        self.canvas.selected=index; self.canvas.active_point=-1; self.canvas.editing=True; self.canvas.finish()

    def close_stroke(self,checked):
        s=self.canvas.current()
        if s:
            self.canvas.remember(); s['closed']=checked; s['build']=0 if checked else 1
            if checked and len(s['points'])>3 and math.dist(s['points'][0],s['points'][-1])<.01: s['points'].pop()
            self.canvas.finish()
    def fill_selected(self):
        if self.active_view():
            try:
                self.live_tool.stop(); selected=cmds.ls(selection=True,objectsOnly=True,long=True) or []
                if not selected: raise ValueError('Click the drawing you want to fill first.')
                node=selected[0]
                if cmds.nodeType(node) in ('mesh','nurbsCurve'): node=cmds.listRelatives(node,parent=True,fullPath=True)[0]
                curves=cmds.listRelatives(node,shapes=True,noIntermediate=True,fullPath=True,type='nurbsCurve') or []
                if cmds.objExists(node+'.bloomStrokeData'):
                    data=json.loads(cmds.getAttr(node+'.bloomStrokeData'))
                    matrix=om.MMatrix(cmds.xform(node,query=True,matrix=True,worldSpace=True))
                    points=[]
                    for value in data['points']:
                        p=om.MPoint(*value)*matrix; points.append((p.x,p.y,p.z))
                    normal=om.MVector(*data['normal'])*matrix
                else:
                    if not curves: raise ValueError('Select a Bloom stroke or an outline curve to fill.')
                    camera=cmds.modelPanel(self.active_view().active_panel(),query=True,camera=True)
                    sel=om.MSelectionList(); sel.add(camera); camera_path=sel.getDagPath(0)
                    if camera_path.node().hasFn(om.MFn.kTransform): camera_path.extendToShape()
                    normal=om.MFnCamera(camera_path).viewDirection(om.MSpace.kWorld)
                    points=[tuple(cmds.pointPosition(cv,world=True)) for cv in cmds.ls(curves[0]+'.cv[*]',flatten=True)]
                if curves: points=[tuple(cmds.pointPosition(cv,world=True)) for cv in cmds.ls(curves[0]+'.cv[*]',flatten=True)]
                result=self.create_persistent_mesh(filled_geometry(points,normal,self.depth.value(),self.depth_segments.value(),self.caps.currentText()),'bloomFilled#')
                self.register_object(result); cmds.select(result,replace=True); self.status.setText('Created a filled solid from the selected outline. Original kept.')
            except Exception as error: self.status.setText('Fill: '+str(error))
            return
        if not self.canvas.current():
            self.status.setText('Select or draw an outline first.'); return
        self.close_stroke(True)
        self.status.setText('Selected outline filled. Other strokes stay unchanged. Make Mesh keeps a copy.')

    def vectorize(self):
        try:
            s=self.canvas.current()
            if not s: raise ValueError('Select a stroke first.')
            pts=sample(s['points'],self.anchors.value(),s['closed']); self.canvas.remember(); s['points']=pts; self.canvas.active_point=-1
            self.canvas.editing=True; self.canvas.finish(); self.status.setText('Drag pink points. Double-click an edge to add a point. Delete removes a selected point.')
        except Exception as error: self.status.setText(str(error))
    def change_preset_category(self,name):
        self.preset_picker.clear(); self.preset_picker.addItems(PRESET_GROUPS.get(name,[]))
        self.animal_part.setEnabled(name=='Animals'); self.animal_style.setEnabled(name=='Animals')

    def preset(self,name):
        animal=self.preset_category.currentText()=='Animals'
        points=animal_outline(name,self.animal_part.currentIndex()==1,self.animal_style.currentIndex()==1) if animal else preset_outline(name)
        self.tabs.setCurrentIndex(0); self.sketch_camera.setCurrentIndex(0)
        settings=self.current_shape_settings()
        self.canvas.remember(); self.canvas.strokes.append({'uid':str(time.time_ns()),'points':points,'closed':True,'build':0,'mesh_settings':settings})
        self.canvas.selected=len(self.canvas.strokes)-1; self.canvas.active_point=-1; self.canvas.editing=True; self.canvas.finish(); self.canvas.fit()
        self.status.setText('Shape added. Its editable Maya object updates automatically; drag points to reshape.')

    def place(self,p,z=0,settings=None):
        settings=settings or {}; units=settings.get('units',self.width_units.value())
        x,y=(p[0]-.5)*units,(.5-p[1])*units; plane=settings.get('plane',self.plane.currentIndex())
        return (z,y,-x) if plane==1 else ((x,z,-y) if plane==2 else (x,y,z))
    def brush_density(self,settings,filled=False,points=0):
        count=max(8,min(1024,int(settings.get('samples',self.resolution.value()))))
        sides=max(3,min(64,int(settings.get('sides',self.tube_sides.value()))))
        layers=max(1,min(128,int(settings.get('layers',self.depth_segments.value()))))
        if settings.get('use_target',False):
            base=settings.get('target',5000)/4**settings.get('subdivision',0)
            quads=settings.get('caps','Triangles')=='Quads • subdivided'
            if filled:
                n=max(3,points); caps=(6 if quads else 2)*(n-2)
                layers=max(1,min(128,round((base-caps)/((4 if quads else 1)*n))))
            elif quads: count=max(8,min(1024,round((base/sides-2)/4+1)))
            else:
                caps=settings.get('caps','Triangles')
                end_faces=0 if caps=='Open' else (2 if caps=='N-gons' else 2*sides)
                count=max(8,min(1024,round((base-end_faces)/sides+1)))
        return count,sides,layers

    def export_curve(self):
        try:
            s=self.canvas.current()
            if not s or len(s['points'])<2: raise ValueError('Select a longer stroke.')
            pts=[self.place(p) for p in s['points']]
            if s['closed']: pts.append(pts[0])
            with chunk('Bloom vector curve'):
                node=cmds.curve(degree=1,point=pts,name='bloomVector#'); cmds.select(node)
            self.register_object(node)
            self.status.setText('Created '+node+'. Its CVs are editable in Maya.')
            return node
        except Exception as error: self.status.setText(str(error))
    def geometry(self,draft=False,stroke=None):
        stroke=stroke if stroke is not None else self.canvas.current()
        if not stroke: raise ValueError('Draw or click a stroke first.')
        settings=stroke.get('mesh_settings',self.current_shape_settings())
        build=stroke.get('build',0 if stroke['closed'] else 1)
        count,sides,layers=self.brush_density(settings,build==0,len(stroke['points']))
        style=settings.get('caps',self.caps.currentText())
        if build==0:
            points=[self.place(p,settings=settings) for p in stroke['points']]
            normal=[(0,0,1),(1,0,0),(0,1,0)][settings.get('plane',self.plane.currentIndex())]
            return filled_geometry(points,normal,settings.get('depth',self.depth.value()),min(layers,4) if draft else layers,style)
        if build==2: raise ValueError('Curves do not have polygon faces.')
        pts=sample(stroke['points'],min(count,64) if draft else count,stroke['closed'])
        path=[self.place(p,settings=settings) for p in pts]
        if stroke['closed']: path.append(path[0])
        vertices,faces=tube(path,settings.get('radius',self.radius.value()),min(sides,12) if draft else sides)
        rings,faces=faces[-2:],faces[:-2]
        for ring in rings: add_cap(vertices,faces,ring,style)
        return split_quads(vertices,faces) if style=='Quads • subdivided' else (vertices,faces)

    def mark_dirty(self,*args):
        self.dirty=True; self.last_edit=time.monotonic()

    def remove_preview(self):
        for key in list(self.previews): self.clear_live_draft(key)
        if self.preview and cmds.objExists(self.preview): cmds.delete(self.preview)
        self.preview=None; self.preview_inputs={}
    def resolve_live_node(self,key):
        uid=self.scene_nodes.get(key)
        nodes=cmds.ls(uid,long=True) if uid else []
        return nodes[0] if nodes else None
    def clear_live_draft(self,key):
        node=self.previews.pop(key,None)
        if node and cmds.objExists(node): cmds.delete(node)
        self.preview_inputs.pop(key,None)
        if key in self.hidden_live:
            old=self.resolve_live_node(key); visible=self.hidden_live.pop(key)
            if old and not cmds.getAttr(old+'.visibility',lock=True): cmds.setAttr(old+'.visibility',visible)
    def final_auto_node(self,key,settings,geometry=None,curvepoints=None,metadata=None):
        old=self.resolve_live_node(key); temp=None; work=None
        levels=int(settings.get('subdivision',0)) if geometry else 0
        if geometry:
            faces=geometry[1]; predicted=len(faces) if not levels else sum(len(f) for f in faces)*4**(levels-1)
            if predicted>200000: raise ValueError('Live result exceeds 200,000 faces. Lower subdivision, samples, sides or target.')
        selected=cmds.ls(selection=True,long=True) or []
        try:
            with transient():
                temp=cmds.curve(degree=1,point=curvepoints,name='bloomBuildTemp#') if curvepoints is not None else create_mesh(geometry,'bloomBuildTemp#')
            with chunk('Bloom live drawing update'):
                work=cmds.duplicate(temp,returnRootsOnly=True,name='bloomDrawing#')[0]
                if geometry:
                    if levels: cmds.polySmooth(work,divisions=levels,continuity=settings.get('roundness',0),constructionHistory=False,keepBorder=True)
                    cmds.polySoftEdge(work,angle=60,constructionHistory=False)
                if old:
                    old_shapes=cmds.listRelatives(old,shapes=True,noIntermediate=True,fullPath=True) or []
                    new_shapes=cmds.listRelatives(work,shapes=True,noIntermediate=True,fullPath=True) or []
                    for shape in new_shapes: cmds.parent(shape,old,shape=True,relative=True)
                    if old_shapes: cmds.delete(old_shapes)
                    cmds.delete(work); work=None; node=old
                else: node=work; work=None
                make_editable(node)
                if metadata is not None:
                    if not cmds.objExists(node+'.bloomStrokeData'): cmds.addAttr(node,longName='bloomStrokeData',dataType='string')
                    cmds.setAttr(node+'.bloomStrokeData',json.dumps(metadata),type='string')
                self.scene_nodes[key]=cmds.ls(node,uuid=True)[0]
            return node
        finally:
            with transient():
                for item in (temp,work):
                    if item and cmds.objExists(item): cmds.delete(item)
                self.clear_live_draft(key)
                valid=[item for item in selected if cmds.objExists(item)]
                if valid: cmds.select(valid,replace=True)
                else: cmds.select(clear=True)
    def draw_live_draft(self,key,stroke,signature):
        if self.preview_inputs.get(key)==signature: return
        settings=stroke.get('mesh_settings',self.current_shape_settings()); build=stroke.get('build',1)
        selected=cmds.ls(selection=True,long=True) or []
        with transient():
            try:
                if build in (0,2):
                    points=[self.place(p,settings=settings) for p in stroke['points']]
                    if stroke['closed'] and len(points)>2: points.append(points[0])
                    new=cmds.curve(degree=1,point=points,name='bloomLiveDraft#')
                else: new=create_mesh(self.geometry(draft=True,stroke=stroke),'bloomLiveDraft#')
                old_draft=self.previews.pop(key,None)
                if old_draft and cmds.objExists(old_draft): cmds.delete(old_draft)
                self.previews[key]=new; self.preview_inputs[key]=signature
                old=self.resolve_live_node(key)
                if old and key not in self.hidden_live and not cmds.getAttr(old+'.visibility',lock=True):
                    self.hidden_live[key]=cmds.getAttr(old+'.visibility'); cmds.setAttr(old+'.visibility',False)
                if old: cmds.xform(new,matrix=cmds.xform(old,query=True,matrix=True,worldSpace=True),worldSpace=True)
                cmds.setAttr(new+'.overrideEnabled',True); cmds.setAttr(new+'.overrideDisplayType',2)
            finally:
                valid=[item for item in selected if cmds.objExists(item)]
                if valid: cmds.select(valid,replace=True)
                else: cmds.select(clear=True)
    def clear_canvas(self):
        self.live_tool.stop()
        if self.active_view():
            nodes=[node for uid in self.viewport_strokes for node in (cmds.ls(uid,long=True) or [])]
            if nodes:
                with chunk('Bloom clear viewport drawings'): cmds.delete(nodes)
            self.viewport_strokes=[]
            for key in list(self.scene_nodes):
                if key.startswith('viewport_'): self.scene_nodes.pop(key,None); self.live_signatures.pop(key,None)
            self.status.setText('Viewport drawings cleared. Other scene objects were kept. Maya Undo restores the drawings.'); return
        self.canvas.remember(); self.canvas.strokes=[]; self.canvas.selected=-1; self.canvas.active_point=-1; self.canvas.editing=False
        self.canvas.finish(); self.update_preview(); self.status.setText('Canvas cleared. Undo restores its drawings.')

    def safe_update_preview(self):
        if self.preview_busy or not self.dirty: return
        gesture=self.canvas.drawing or self.canvas.dragging or self.canvas.moving_stroke
        if not gesture and time.monotonic()-self.last_edit<.2: return
        self.preview_busy=True
        try: self.update_preview()
        except Exception as error:
            self.dirty=False; self.status.setText('Live drawing: '+str(error))
        finally: self.preview_busy=False

    def update_preview(self):
        if not self.dirty: return
        gesture=self.canvas.drawing or self.canvas.dragging or self.canvas.moving_stroke
        if not gesture: self.dirty=False
        active=set(); errors=[]; total_faces=0
        pending=getattr(self,'pending_viewport_update',None)
        if pending and not gesture:
            self.pending_viewport_update=None; key,settings,data=pending
            output=data.get('output',0)
            geometry=None; curvepoints=None
            if output==1: curvepoints=sample(data['points'],settings['samples'],False)
            else: geometry=self.live_geometry(data['points'],om.MVector(*data['normal']),False,settings,output)
            data['settings']=settings
            self.final_auto_node(key,settings,geometry,curvepoints,data)
        for index,stroke in enumerate(self.canvas.strokes):
            key=stroke.setdefault('uid',str(time.time_ns())+'_'+str(index)); active.add(key)
            if len(stroke['points'])<2: continue
            signature=hash(repr((stroke['points'],stroke['closed'],stroke.get('build'),stroke.get('mesh_settings',{}))))
            if gesture:
                if index==self.canvas.selected: self.draw_live_draft(key,stroke,signature)
                continue
            node=self.resolve_live_node(key)
            if self.live_signatures.get(key)==signature and node:
                with transient(): self.clear_live_draft(key)
                continue
            try:
                settings=stroke.get('mesh_settings',self.current_shape_settings()); build=stroke.get('build',1)
                points=[self.place(p,settings=settings) for p in stroke['points']]
                curvepoints=None; geometry=None
                if build==2:
                    curvepoints=sample(points,settings.get('samples',96),stroke['closed'])
                    if stroke['closed']: curvepoints.append(curvepoints[0])
                else: geometry=self.geometry(stroke=stroke)
                normal=[(0,0,1),(1,0,0),(0,1,0)][settings.get('plane',0)]
                metadata={'points':points,'normal':normal,'output':{0:2,1:0,2:1}[build],'settings':settings}
                node=self.final_auto_node(key,settings,geometry,curvepoints,metadata)
                self.live_signatures[key]=signature
                if geometry: total_faces+=cmds.polyEvaluate(node,face=True)
            except Exception as error:
                with transient(): self.clear_live_draft(key)
                errors.append(str(error))
        if not gesture:
            for key in list(self.scene_nodes):
                if key.startswith('viewport_') or key in active: continue
                node=self.resolve_live_node(key)
                if node:
                    with chunk('Bloom remove canvas drawing'): cmds.delete(node)
                self.scene_nodes.pop(key,None); self.live_signatures.pop(key,None)
            for key in list(self.previews):
                if key not in active:
                    with transient(): self.clear_live_draft(key)
        self.stats.setText('Live drawings: {} • changes update Maya automatically'.format(len(self.canvas.strokes)))
        if errors: self.status.setText('Live drawing: '+errors[0])
        elif not gesture and total_faces: self.shape_counts.setText('Updated live result: {:,} faces'.format(total_faces))

    def commit_all(self):
        original=self.canvas.selected
        if not self.canvas.strokes: self.status.setText('Draw some strokes first.'); return
        created=[]
        try:
            for index in range(len(self.canvas.strokes)):
                self.canvas.selected=index
                node=self.commit()
                if node: created.append(node)
            if created:
                cmds.select(created,replace=True)
                try: cmds.viewFit('persp',allObjects=False)
                except RuntimeError: pass
                self.status.setText('Created '+str(len(created))+' independent meshes. Overlapping parts are not fused.')
        finally:
            self.canvas.selected=original; self.canvas.finish()
    def commit(self):
        self.dirty=True; self.update_preview()
        stroke=self.canvas.current()
        return self.resolve_live_node(stroke.get('uid')) if stroke else None

    def add_view_draw_buttons(self):
        for view in self.views: view.studio=self
        self.refresh_scene_objects()


    def change_sketch_camera(self,index):
        self.sketch_stack.setCurrentIndex(index)
        self.native_edit=False; self.live_tool.stop(); cmds.setToolTo('selectSuperContext')
        if index==1:
            self.status.setText('Drag empty space to draw. Click an object to select it. Alt + mouse navigates. Shift-drag draws over another object.')
            QtCore.QTimer.singleShot(120,self.sketch_perspective.ensure_ready)
    def center_selected(self):
        try:
            self.live_tool.stop(); objects=[]
            for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
                if cmds.nodeType(item) in ('mesh','nurbsCurve'): item=cmds.listRelatives(item,parent=True,fullPath=True)[0]
                if cmds.objectType(item,isAType='transform') and item not in objects: objects.append(item)
            objects=[item for item in objects if not any(item.startswith(parent+'|') for parent in objects if parent!=item)]
            if not objects: raise ValueError('Select an object to center.')
            bounds=cmds.exactWorldBoundingBox(objects); center=[(bounds[i]+bounds[i+3])/2 for i in range(3)]
            with chunk('Bloom center selection'):
                cmds.move(-center[0],-center[1],-center[2],objects,relative=True,worldSpace=True)
            self.status.setText('Selection centered at Maya origin. New drawings use the scene-origin plane by default.')
        except Exception as error: self.status.setText(str(error))
    def sketch_undo(self):
        if self.active_view(): cmds.undo()
        else: self.canvas.undo()
    def sketch_redo(self):
        if self.active_view(): cmds.redo()
        else: self.canvas.redo()
    def fit_sketch_view(self):
        if self.sketch_stack.currentIndex()==1: self.sketch_perspective.frame()
        else: self.canvas.fit()

    def make_brush_widget(self):
        container=QtWidgets.QWidget(); layout=QtWidgets.QVBoxLayout(container); layout.setContentsMargins(0,0,0,0)
        spin=self.number(self.brush_size.minimum(),self.brush_size.maximum(),self.brush_size.value()); spin.setSuffix(' units')
        slider=QtWidgets.QSlider(QtCore.Qt.Horizontal); slider.setRange(0,1000)
        def sync(value):
            slider.blockSignals(True); slider.setValue(round(1000*math.log10(max(.002,value)/.002)/5)); slider.blockSignals(False)
        spin.valueChanged.connect(self.brush_size.setValue); self.brush_size.valueChanged.connect(spin.setValue); self.brush_size.valueChanged.connect(sync)
        slider.valueChanged.connect(lambda value:self.brush_size.setValue(.002*10**(5*value/1000)))
        sync(self.brush_size.value()); layout.addWidget(spin); layout.addWidget(slider); container.setToolTip('Brush diameter in Maya units. [ makes it smaller; ] makes it larger. Applies to new tube strokes.'); return container
    def install_brush_shortcuts(self):
        shortcut_class=getattr(QtGui,'QShortcut',None) or QtWidgets.QShortcut
        self.smaller_brush=shortcut_class(QtGui.QKeySequence('['),self); self.larger_brush=shortcut_class(QtGui.QKeySequence(']'),self)
        for shortcut in [self.smaller_brush,self.larger_brush]: shortcut.setContext(QtCore.Qt.WidgetWithChildrenShortcut)
        self.smaller_brush.activated.connect(lambda:self.brush_size.setValue(self.brush_size.value()*.8)); self.larger_brush.activated.connect(lambda:self.brush_size.setValue(self.brush_size.value()*1.25))
    def surface_label_widget(self):
        label=QtWidgets.QLabel('No character / surface target'); label.setWordWrap(True); self.surface_labels.append(label); return label
    def set_paintable(self):
        try:
            self.native_edit=False; cmds.selectMode(object=True); cmds.setToolTo('selectSuperContext')
            self.live_tool.stop(); shapes=[]
            for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
                if cmds.nodeType(item)=='mesh': candidates=[item]
                else: candidates=(cmds.listRelatives(item,allDescendents=True,fullPath=True,type='mesh') or [])+(cmds.listRelatives(item,shapes=True,fullPath=True,type='mesh') or [])
                for shape in candidates:
                    if not cmds.getAttr(shape+'.intermediateObject') and shape not in shapes: shapes.append(shape)
            if not shapes: raise ValueError('Select the character mesh or its group first.')
            self.paintable_shapes=[cmds.ls(shape,uuid=True)[0] for shape in shapes]; self.live_surface.setChecked(True)
            text='Paintable target: {} mesh surface(s)'.format(len(shapes))
            for label in self.surface_labels: label.setText(text)
            self.status.setText(text+'. Choose Draw Tubes or Draw Curves, then draw on the character. Strokes follow its current pose as separate geometry; they are not automatically rigged.')
        except Exception as error: self.status.setText(str(error))
    def clear_paintable(self):
        self.live_tool.stop(); self.paintable_shapes=[]; self.live_surface.setChecked(False)
        for label in self.surface_labels: label.setText('No character / surface target')
    def paintable_paths(self):
        paths=[]
        for uid in self.paintable_shapes:
            for node in cmds.ls(uid,long=True) or []:
                selection=om.MSelectionList(); selection.add(node); paths.append(selection.getDagPath(0))
        return paths
    def mirror_control(self,source):
        if isinstance(source,QtWidgets.QComboBox):
            w=self.combo([source.itemText(i) for i in range(source.count())]); w.setCurrentIndex(source.currentIndex())
            w.currentIndexChanged.connect(source.setCurrentIndex); source.currentIndexChanged.connect(w.setCurrentIndex)
        elif isinstance(source,QtWidgets.QCheckBox):
            w=QtWidgets.QCheckBox(source.text()); w.setChecked(source.isChecked())
            w.toggled.connect(source.setChecked); source.toggled.connect(w.setChecked)
        elif isinstance(source,QtWidgets.QDoubleSpinBox):
            w=self.number(source.minimum(),source.maximum(),source.value()); w.setDecimals(source.decimals())
            w.valueChanged.connect(source.setValue); source.valueChanged.connect(w.setValue)
        else:
            w=self.integer(source.minimum(),source.maximum(),source.value())
            w.valueChanged.connect(source.setValue); source.valueChanged.connect(w.setValue)
        return w

    def select_edit(self,keep_message=False):
        self.native_edit=False; self.focus_active_view()
        self.live_tool.stop(); cmds.selectMode(object=True); cmds.selectType(polymesh=True,nurbsCurve=True); cmds.setToolTo('selectSuperContext')
        if not keep_message: self.status.setText('Select / Edit: click any completed object. Use Edit Vertices / Curve Points for shape editing.')
    def scene_transform(self,context):
        if not self.active_view():
            self.canvas.editing=True; self.canvas.update(); self.status.setText('Drag any selected outline to move it, or a point to reshape it.'); return
        self.native_edit=True; self.focus_active_view()
        self.live_tool.stop()
        try: cmds.setToolTo(context)
        except RuntimeError as error: self.status.setText(str(error))
    def edit_scene_points(self):
        try:
            self.native_edit=True; self.focus_active_view()
            self.live_tool.stop(); objects=cmds.ls(selection=True,objectsOnly=True,long=True) or []
            parents=[]
            for obj in objects:
                if cmds.nodeType(obj) in ('mesh','nurbsCurve'):
                    obj=cmds.listRelatives(obj,parent=True,fullPath=True)[0]
                shapes=cmds.listRelatives(obj,shapes=True,noIntermediate=True,fullPath=True) or []
                if any(cmds.nodeType(shape) in ('mesh','nurbsCurve') for shape in shapes): parents.append(obj)
            if not parents: raise ValueError('Select a mesh or curve first.')
            cmds.hilite(parents,replace=True); cmds.selectMode(component=True); cmds.selectType(allComponents=False); cmds.selectType(polymeshVertex=True,controlVertex=True); cmds.select(clear=True); cmds.setToolTo('selectSuperContext')
            self.status.setText('Click a vertex or curve control point, then press W to move it. Select / Edit returns to objects.')
        except Exception as error: self.status.setText(str(error))
    def update_selection_stats(self):
        try:
            mesh,_=selected_mesh(); text='Selected: {}\n{:,} faces • {:,} vertices'.format(mesh.split('|')[-1],cmds.polyEvaluate(mesh,face=True),cmds.polyEvaluate(mesh,vertex=True))
        except Exception: text='Select a mesh to see its actual face count.'
        for label in self.scene_stats_labels: label.setText(text)
    def register_object(self,node):
        ids=cmds.ls(node,uuid=True) or []
        if ids and ids[0] not in self.created_nodes: self.created_nodes.append(ids[0])
        self.refresh_scene_objects()
    def refresh_scene_objects(self):
        nodes=cmds.ls('bloomStroke*','bloomMesh*','bloomVector*','bloomCup*','bloomJoined*','bloomRetopo*','bloomDense*',long=True,type='transform') or []
        for uid in self.created_nodes:
            nodes.extend(cmds.ls(uid,long=True) or [])
        unique={}
        for node in nodes:
            shapes=cmds.listRelatives(node,shapes=True,noIntermediate=True,fullPath=True) or []
            if any(cmds.nodeType(shape) in ('mesh','nurbsCurve') for shape in shapes):
                ids=cmds.ls(node,uuid=True) or []
                if ids: unique[ids[0]]=node
        for picker in self.scene_pickers:
            old=picker.currentData(); picker.blockSignals(True); picker.clear()
            for uid,node in sorted(unique.items(),key=lambda item:item[1]): picker.addItem(node.split('|')[-1],uid)
            index=picker.findData(old)
            if index>=0: picker.setCurrentIndex(index)
            picker.blockSignals(False)
    def select_list_object(self,picker):
        uid=picker.currentData(); nodes=cmds.ls(uid,long=True) if uid else []
        if not nodes: self.status.setText('Choose an existing object or refresh the list.'); return
        self.select_edit(); make_editable(nodes[0]); cmds.select(nodes[0],replace=True)
    def apply_face_target(self,node,settings=None):
        # Live tools approximate face counts through density. Native retopology is
        # available only as an explicit action in the Retopology tab.
        return ''

    def retopo_selected_copy(self):
        self.select_edit(); self.retopo_source=None; self.capture_source()
        if self.retopo_source:
            self.target_faces.setValue(self.face_target.value()); self.retopologize()
    def live_geometry(self,points,normal,draft=False,settings=None,output=None):
        settings=settings if settings is not None else getattr(self.live_tool,'settings',self.current_shape_settings())
        output=self.live_output.currentIndex() if output is None else output
        count,sides,layers=self.brush_density(settings,output==2,len(points))
        style=settings.get('caps','Triangles')
        if output==2: return filled_geometry(points,normal,settings.get('depth',.4),min(layers,4) if draft else layers,style)
        clean=[]
        for point in points:
            if not clean or math.dist(point,clean[-1])>1e-7: clean.append(point)
        sampled=sample(clean,min(count,64) if draft else count,False)
        vertices,faces=tube(sampled,settings.get('radius',.12),min(sides,12) if draft else sides)
        rings,faces=faces[-2:],faces[:-2]
        for ring in rings: add_cap(vertices,faces,ring,style)
        return split_quads(vertices,faces) if style=='Quads • subdivided' else (vertices,faces)

    def build_live(self):
        # Shared by all drawing views; the essential controls live in the top Tools bar.
        self.live_output=self.combo(['Tube','Filled shape','Curve'])
        # Keep the established internal indices: Tube=0, Curve=1, Filled=2.
        self.live_output.clear(); self.live_output.addItems(['Tube','Curve','Filled shape'])
        self.live_surface=QtWidgets.QCheckBox('Draw on surface')
        self.live_plane=self.combo(['Camera distance','Scene origin','Selected object center']); self.live_plane.setCurrentIndex(1)
        self.live_distance=self.number(.1,100000,20); self.live_spacing=self.number(.001,100,.025); self.surface_offset=self.number(-10,10,.02)
        self.keep_drawing=QtWidgets.QCheckBox('Keep drawing'); self.keep_drawing.setChecked(True)
        self.live_output.currentIndexChanged.connect(self.change_live_output)
        self.live_surface.toggled.connect(self.surface_mode_changed)

    def change_live_output(self,index):
        self.native_edit=False; cmds.selectMode(object=True); cmds.setToolTo('selectSuperContext')
        self.live_tool.stop()
        self.canvas.mode='Continuous fill' if index==2 else 'Sketch'
        self.canvas.new_build={0:1,1:2,2:0}[index]
        if index==2: self.live_surface.setChecked(False)
        self.status.setText(['Tube: drag a round stroke. Click an existing drawing to edit.','Curve: drag a line with editable control points.','Filled shape: draw the outline and release to fill its whole interior.'][index])

    def start_live(self):
        try: self.live_tool.activate()
        except Exception as error: self.status.setText(str(error))
    def create_persistent_mesh(self,geometry,name):
        temp=None
        try:
            with transient(): temp=create_mesh(geometry,'bloomBuildTemp#')
            with chunk('Bloom create part'):
                node=cmds.duplicate(temp,returnRootsOnly=True,name=name)[0]; make_editable(node)
            return node
        finally:
            with transient():
                if temp and cmds.objExists(temp): cmds.delete(temp)
    def mesh_parts_selected(self):
        parts=[]
        for item in cmds.ls(selection=True,objectsOnly=True,long=True) or []:
            if cmds.nodeType(item)=='mesh': shapes=[item]
            else: shapes=cmds.listRelatives(item,shapes=True,noIntermediate=True,fullPath=True,type='mesh') or []
            for shape in shapes:
                parent=cmds.listRelatives(shape,parent=True,fullPath=True)[0]
                if parent not in parts: parts.append(parent)
        if len(parts)<2: raise ValueError('Select at least two polygon parts. Use Shift-click to select the cup body and handle.')
        return parts
    def join_selected(self,fuse=False):
        copies=[]; outputs=[]; result=None; parts=[]; visibility={}
        try:
            self.live_tool.stop(); parts=self.mesh_parts_selected(); cmds.waitCursor(state=True)
            with chunk('Bloom fuse parts' if fuse else 'Bloom combine parts'):
                try:
                    for part in parts:
                        duplicate=cmds.duplicate(part,returnRootsOnly=True,name='bloomJoinWork#')[0]
                        if cmds.listRelatives(duplicate,parent=True): duplicate=cmds.parent(duplicate,world=True)[0]
                        cmds.setAttr(duplicate+'.visibility',True); copies.append(duplicate)
                    if fuse:
                        result=copies[0]
                        for part in copies[1:]:
                            values=cmds.polyCBoolOp(result,part,operation=1)
                            if not values: raise ValueError('Maya returned no union result.')
                            result=values[0]; outputs.append(result); cmds.delete(result,constructionHistory=True)
                        if cmds.polyEvaluate(result,shell=True)!=1:
                            raise ValueError('The result still has separate shells. Move the parts so solid surfaces overlap, then fuse again. Combine does not require overlap.')
                    else: result=cmds.polyUnite(copies,constructionHistory=False)[0]; outputs.append(result)
                    result=cmds.rename(result,'bloomJoined#'); outputs.append(result); make_editable(result)
                    for part in parts:
                        plug=part+'.visibility'
                        if cmds.objExists(plug) and not cmds.getAttr(plug,lock=True):
                            visibility[part]=cmds.getAttr(plug); cmds.setAttr(plug,False)
                    for node in copies+outputs:
                        if node!=result and cmds.objExists(node): cmds.delete(node)
                    cmds.select(result,replace=True)
                except Exception:
                    for node in copies+outputs+([result] if result else []):
                        if node and cmds.objExists(node): cmds.delete(node)
                    for part,value in visibility.items():
                        if cmds.objExists(part): cmds.setAttr(part+'.visibility',value)
                    raise
            self.register_object(result)
            self.status.setText(('Fused into one connected mesh: ' if fuse else 'Combined into one object; separate shells may remain: ')+result+' • {:,} faces. Source parts are retained and hidden where possible.'.format(cmds.polyEvaluate(result,face=True)))
            return result
        except Exception as error:
            self.status.setText('Could not join parts: '+str(error))
            valid=[part for part in parts if cmds.objExists(part)]
            if valid: cmds.select(valid,replace=True)
            return None
        finally: cmds.waitCursor(state=False)
    def create_cup_starter(self):
        try:
            self.live_tool.stop(); radius=self.cup_radius.value(); height=self.cup_height.value(); wall=self.cup_wall.value(); base=self.cup_base.value()
            geometry=cup_geometry(radius,height,wall,base,self.cup_sides.value(),self.cup_segments.value())
            handle_radius=self.cup_handle_radius.value(); reach=self.cup_handle_reach.value()
            points=[]
            for i in range(65):
                a=math.pi*i/64; points.append((radius-handle_radius*.35+reach*math.sin(a),height*.5+height*.28*math.cos(a),0))
            with chunk('Bloom teacup starter'):
                body=self.create_persistent_mesh(geometry,'bloomCupBody#')
                handle=self.create_persistent_mesh(tube(points,handle_radius,self.tube_sides.value()),'bloomCupHandle#')
                for node in [body,handle]: cmds.polySoftEdge(node,angle=60,constructionHistory=False); self.register_object(node)
                cmds.select([body,handle],replace=True)
                if self.cup_fuse.isChecked():
                    result=self.join_selected(fuse=True)
                    if result:
                        note=self.apply_face_target(result); self.status.setText('Created a hollow teacup with a connected handle • {:,} faces. '.format(cmds.polyEvaluate(result,face=True))+note)
                else: self.status.setText('Cup body and handle created as two editable parts. Shift-select both and Fuse Selected Solids when ready.')
            self.tabs.setCurrentIndex(3)
            try: cmds.viewFit('persp',allObjects=False)
            except RuntimeError: pass
        except Exception as error: self.status.setText('Teacup starter: '+str(error))
    def build_assembly(self):
        page=QtWidgets.QWidget(); outer=QtWidgets.QVBoxLayout(page); scroll=QtWidgets.QScrollArea(); scroll.setWidgetResizable(True); panel=QtWidgets.QWidget(); layout=QtWidgets.QVBoxLayout(panel); scroll.setWidget(panel); outer.addWidget(scroll)
        label=QtWidgets.QLabel('Build complex objects from parts. Combine collects parts into one Maya object. Fuse uses a boolean union to connect overlapping solids. It does not bridge gaps or infer hidden shapes.'); label.setWordWrap(True); layout.addWidget(label)
        self.group(layout,'Join selected parts',[('',self.button('Combine Selected Parts',lambda checked=False:self.join_selected(False))),('',self.button('Fuse Selected Solids',lambda checked=False:self.join_selected(True)))])
        self.cup_radius=self.number(.1,100,1.5); self.cup_height=self.number(.1,100,2.4); self.cup_wall=self.number(.01,20,.15); self.cup_base=self.number(.01,20,.18)
        self.cup_handle_reach=self.number(.05,100,.8); self.cup_handle_radius=self.number(.01,10,.12); self.cup_sides=self.integer(16,256,96); self.cup_segments=self.integer(1,64,16)
        self.cup_fuse=QtWidgets.QCheckBox('Fuse teacup body and handle'); self.cup_fuse.setChecked(True)
        self.group(layout,'Hollow teacup starter',[('Radius',self.cup_radius),('Height',self.cup_height),('Wall thickness',self.cup_wall),('Bottom thickness',self.cup_base),('Handle reach',self.cup_handle_reach),('Handle radius',self.cup_handle_radius),('Radial segments',self.cup_sides),('Wall segments',self.cup_segments),('',self.cup_fuse),('',self.button('Create Teacup Starter',self.create_cup_starter))])
        note=QtWidgets.QLabel('The starter creates an actual hollow body, then adds a tube handle. You can edit both parts with Maya tools or draw your own handle and fuse it. Intersecting inputs and valid closed surfaces give the most reliable union.'); note.setWordWrap(True); layout.addWidget(note); layout.addStretch(); self.tabs.addTab(page,'Assemble Parts')
    def build_help(self):
        page=QtWidgets.QWidget(); layout=QtWidgets.QVBoxLayout(page); browser=QtWidgets.QTextBrowser()
        browser.setStyleSheet('QTextBrowser {background:#FAF5FF;color:#34234F;border:none;padding:12px;}')
        browser.setHtml('''<h2>Sketch Bloom Live Studio ✿</h2>
        <p><b>Draw and edit directly. Maya objects update automatically.</b> Drag empty space to draw. Click an existing drawing to select and reshape it. No save/commit button is needed.</p>
        <h3>Tube, Curve and Filled shape</h3><p>Tube makes a round mesh stroke. Curve makes an editable line without faces. Filled shape closes and fills on release. While dragging, previews stay lightweight; the chosen subdivision applies when you release.</p>
        <h3>Brush &amp; tool polygons</h3><p>Path samples and Tube sides control the brush mesh density. Subdivision adds polygons; Roundness controls smoothing. Depth segments controls filled walls. These settings affect the selected drawing and the active brush for your next strokes. Aim for polygon count adjusts sampling or wall density to approach the target; counts are approximate. Curve strokes have control points rather than polygons.</p>
        <p>Live results are limited to 200,000 faces to keep interaction manageable. Native retopology is an explicit action in its own tab.</p>
        <h3>Edit and clear</h3><p>Click a 2D drawing to reveal pink handles. Drag a handle to reshape, or drag the outline/interior to move it. Shift-drag starts a stroke over an existing one. Double-click an edge to add a point. Delete removes a point or drawing. Clear Canvas removes this canvas's drawings; Undo restores them. In a 3D view, Clear Canvas removes viewport strokes made by this Studio, preserving unrelated objects.</p>
        <h3>3D and surface drawing</h3><p>Click an object to select it. Move or Edit points uses Maya tools; Q returns to drawing. Alt + mouse navigates. Use selected surface captures a character mesh/group for Tube or Curve projection on its current pose. Clear surface returns to plane drawing. Strokes are separate geometry and are not automatically rigged.</p>
        <h3>Shapes, saving and windows</h3><p>Animals offer heads/full bodies and stylized/natural silhouettes. Shapes are outline starters rather than complete 3D animals. Every finished drawing is kept automatically and saves with your Maya scene. Closing the Studio leaves the finished objects in Maya. Use the title bar or edges to resize; F11 toggles fullscreen.</p>''')
        layout.addWidget(browser); self.tabs.addTab(page,'Help')

    def build_retopo(self):
        page=QtWidgets.QWidget(); layout=QtWidgets.QVBoxLayout(page)
        label=QtWidgets.QLabel('Select an existing polygon object in Maya, then capture it below. Retopology and subdivision create separate copies for comparison.'); label.setWordWrap(True); layout.addWidget(label)
        self.source_label=QtWidgets.QLabel('No source selected'); layout.addWidget(self.source_label); layout.addWidget(self.button('Use Selected Object',self.capture_source))
        self.target_faces=self.integer(100,1000000,5000); self.keep_edges=QtWidgets.QCheckBox('Preserve hard edges'); self.keep_edges.setChecked(True)
        self.group(layout,'Automatic retopology',[('Target quads',self.target_faces),('',self.keep_edges),('',self.button('Retopologize Copy',self.retopologize))])
        self.refine_levels=self.integer(1,4,1); self.refine_round=self.number(0,1,0)
        self.group(layout,'High polygon copy',[('Subdivision levels',self.refine_levels),('Roundness',self.refine_round),('',self.button('Subdivide Copy',self.refine_copy))])
        hint=QtWidgets.QLabel('Retopology calls Maya’s native polyRetopo when available. Target counts are approximate. Dense meshes can take time. New topology does not transfer UVs, skinning, or blend shapes.\n\nFor manual retopology, use Maya’s Modeling Toolkit / Quad Draw on your source surface.')
        hint.setWordWrap(True); layout.addWidget(hint); layout.addWidget(self.button('Open Maya Modeling Toolkit',self.open_toolkit)); layout.addStretch(); self.tabs.addTab(page,'Retopology')
    def capture_source(self):
        try:
            source,_=selected_mesh(); self.retopo_source=(cmds.ls(source,uuid=True) or [None])[0]; self.source_label.setText(source)
        except Exception as error: self.status.setText(str(error))
    def source_mesh(self):
        if not self.retopo_source: raise ValueError('Use Selected Object first.')
        nodes=cmds.ls(self.retopo_source,long=True) or []
        if not nodes: raise ValueError('The source no longer exists. Select it again.')
        return nodes[0]
    def retopologize(self):
        result=None
        try:
            source=self.source_mesh(); cmds.waitCursor(state=True)
            with chunk('Bloom retopology copy'):
                try:
                    result=cmds.duplicate(source,returnRootsOnly=True,name='bloomRetopo#')[0]; cmds.delete(result,constructionHistory=True)
                    cmds.polyRetopo(result,targetFaceCount=self.target_faces.value(),preserveHardEdges=self.keep_edges.isChecked()); cmds.select(result)
                except Exception:
                    if result and cmds.objExists(result): cmds.delete(result)
                    raise
            self.register_object(result)
            self.status.setText('Retopology copy: {} • {:,} faces. Source kept.'.format(result,cmds.polyEvaluate(result,face=True)))
        except Exception as error: self.status.setText('Retopology could not run in this Maya installation: '+str(error))
        finally: cmds.waitCursor(state=False)
    def refine_copy(self):
        result=None
        try:
            source=self.source_mesh()
            if cmds.polyEvaluate(source,face=True)*4**self.refine_levels.value()*2>2000000: raise ValueError('Reduce subdivision levels for this source.')
            cmds.waitCursor(state=True)
            with chunk('Bloom subdivide copy'):
                try:
                    result=cmds.duplicate(source,returnRootsOnly=True,name='bloomDense#')[0]; cmds.delete(result,constructionHistory=True)
                    cmds.polySmooth(result,divisions=self.refine_levels.value(),continuity=self.refine_round.value(),constructionHistory=False); cmds.select(result)
                except Exception:
                    if result and cmds.objExists(result): cmds.delete(result)
                    raise
            self.register_object(result)
            self.status.setText('Created {} • {:,} faces.'.format(result,cmds.polyEvaluate(result,face=True)))
        except Exception as error: self.status.setText(str(error))
        finally: cmds.waitCursor(state=False)
    def open_toolkit(self):
        try:
            from maya import mel
            mel.eval('ModelingToolkit;')
        except Exception as error: self.status.setText('Open Modeling Toolkit from Maya’s toolbar. '+str(error))
    def closeEvent(self,event):
        self.timer.stop(); self.selection_timer.stop(); self.live_tool.stop()
        with transient(): self.remove_preview()
        for view in self.views: view.cleanup()
        # QDialog.closeEvent calls reject(); our Escape handler intentionally does not
        # close drafts, so accept the explicit title-bar Close action directly.
        event.accept()

def show():
    global bloom_pro_window
    for widget in QtWidgets.QApplication.topLevelWidgets():
        if widget.objectName() in ('SketchBloomWindow','SketchBloomAdvancedWindow','SketchBloomStudioWindow','SketchBloomProWindow'):
            widget.close(); widget.deleteLater()
    parent=wrapInstance(int(OpenMayaUI.MQtUtil.mainWindow()),QtWidgets.QWidget)
    bloom_pro_window=BloomPro(parent); bloom_pro_window.show(); return bloom_pro_window

if __name__=='__main__':
    show()
```
