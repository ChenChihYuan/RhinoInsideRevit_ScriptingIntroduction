# RhinoInsideRevit_ScriptingIntroduction
This folder is for the TokyoAEC weekly session. 
The goal of this weekly session is to introduce how we can use the RhinoInside to script in Grasshopper and link to Revit.

# Official Document
https://www.rhino3d.com/inside/revit/1.0/guides/rir-csharp

Two `.dll` files are vital for developing in Revit.
- RevitAPI.dll
- RevitAPIUI.dll


However there are currently located in different folders, if you have multiple Revit version in your local machine.
You can find those `.dll` files in the directory `%PROGRAMFILES%/Autodesk/Revit XXXX`

- `XXXX` is the Revit version you want to use.

For example,
![image](https://user-images.githubusercontent.com/13364367/143612841-090a3929-cdc3-4491-a775-63c47230d25b.png)


One `.dll` file for **RhinoInsideRevit**
- RhinoInside.Revit.dll

Typically, the file location will be placed in `%PROGRAMDATA%/Autodesk/Revit/XXXX/RhinoInside.Revit`

Also, `XXXX` means the version you are using.

```cs
using DB = Autodesk.Revit.DB;
using UI = Autodesk.Revit.UI;

using RhinoInside.Revit;
using RhinoInside.Revit.Convert.Geometry;
```
---
We can follow the official document to create our own `Custom User Component`

File -> Create User Object -> OK

---

# Example

This example will introduce how we can create a sphere and bring it into the Revit.

## Make a sphere in with RhinoCommon

```cs
private void RunScript(double Radius, ref object Sphere)
  {
    Sphere = new Sphere(Point3d.Origin, Radius);
  }
```

## Bake into Revit

Although currently on the offical documentation, McNeel suggests to use `EnqueueAction` to execute the `transaction`, it is both out of date and will not be supported in the future.

**Reference:** 
- python: https://discourse.mcneel.com/t/enqueue-action-not-working-action-not-defined/108634/2
- C#: https://discourse.mcneel.com/t/rhino-inside-revit/126351/2

> In next release EnqueueAction will be obsolete and removed in the future, so please don’t relay on it. -Kike Garcia

So, how exactly can we do transaction? 

Following up from the reference post, you may find a .gh file, inside there's a python script shows how to implement `transaction` in python script.

Here, I provide the C# version, I tried and it worked
```cs
using(var txx = new DB.Transaction(Revit.ActiveDBDocument, Component.NickName))
    {
      txx.Start();
      // Changes you want to make in Revit      
      txx.Commit();
    }
```

Few things can be noticed
- using block
- transaction is stored as an object
- has `.Start()` and `.End()`

If you want to know more about transaction, you can check the official RevitAPI doc.
https://www.revitapidocs.com/2015/5eb09fb7-8ef2-f977-f1a1-5ee966d7e97c.htm


```cs
private void RunScript(double Radius, bool bake, ref object Sphere)
  {
    if(bake == false) return;

    var doc = Revit.ActiveDBDocument;
    using(var tx = new DB.Transaction(doc, Component.NickName))
    {
      tx.Start();
      Sphere = CreateSphere(doc, Radius);
      tx.Commit();
    }
  }

  // <Custom additional code> 
  private DB.DirectShape CreateSphere(DB.Document doc, double radius)
  {
    var sphere = new Rhino.Geometry.Sphere(Rhino.Geometry.Point3d.Origin, radius);
    var brep = sphere.ToBrep();

    var revitCategory = new DB.ElementId((int) DB.BuiltInCategory.OST_GenericModel); 

    var ds = DB.DirectShape.CreateElement(doc, revitCategory);
    var shapeList = new List<DB.GeometryObject>() { brep.ToSolid() };
    ds.AppendShape(shapeList);
    return ds;
  }
```

Some things worth to notice here, we are using `DirectShape` class to bring Grasshopper Geometry(`Brep`) into Revit.
Apparently, `DirectShape`'s category is under `DB.BuiltInCategory.OST_GenericModel`.

Some previous thoughts about `direct shape`, maybe worth to be discussed.
https://chenchihyuan.github.io/2021/10/06/DirectShape-Introduction/

According to the RevitAPI doc
> This class is used to store externally created geometric shapes. Primary intended use is for importing shapes from other data formats such as IFC or STEP. A DirectShape object may be assigned a category. That will affect how that object is displayed in Revit.

I guess, IFC and RhinoInside are both using `directShape` as one entry point to connect with Revit.

---

# Exercise 1 - Direct Shape + NodeInCode (from the Official Document)

Apparently, we can link components using nodeInCode.
Here is the example.

More about NodeInCode:
https://chenchihyuan.github.io/2020/05/03/NodeInCode/
https://discourse.mcneel.com/t/rhino-nodeincode-namespace/38622


# Exercise 2 - make Revit Rebar in Grasshopper

```cs
private void RunScript(object HostElement, bool Trigger, List<Curve> crvs, Vector3d norm, object BarDiameter, ref object rebar_result)
  {
    if(Trigger == false) return;

    host = HostElement as DB.Element;

    using(var txx = new DB.Transaction(Revit.ActiveDBDocument, Component.NickName))
    {
      txx.Start();
      var rebar = CreateRebar(Revit.ActiveDBDocument, host, crvs, norm.ToXYZ());
      // delete the hook

      var invId = DB.ElementId.InvalidElementId;
      rebar.SetHookTypeId(0, invId);
      rebar.SetHookTypeId(1, invId);

      // output
      rebar_result = rebar;
      txx.Commit();
    }

  }

  // <Custom additional code> 
  DB.Element host;
  private DB.Structure.Rebar CreateRebar(DB.Document doc, DB.Element hostElement, List<Curve> crvs, DB.XYZ normal){
    // Create the line rebar
    IList<DB.Curve> curves = crvs.Select(crv => crv.ToCurve()).ToList();
    // creating input parameters
    var barType = DB.Structure.RebarBarType.Create(Revit.ActiveDBDocument);

    var hookType = DB.Structure.RebarHookType.Create(Revit.ActiveDBDocument, Math.PI, 90);
    var rebar = DB.Structure.Rebar.CreateFromCurves(Revit.ActiveDBDocument, Autodesk.Revit.DB.Structure.RebarStyle.Standard, barType, hookType, hookType, hostElement as DB.Element, normal, curves, DB.Structure.RebarHookOrientation.Right, DB.Structure.RebarHookOrientation.Left, true, true);
    // if the ActiveView is View3D
    if(doc.ActiveView as DB.View3D != null)
      rebar.SetSolidInView(doc.ActiveView as DB.View3D, true);
    return rebar;
  }
```

---
# Inspecting Revit

Due to there're some difference between each version of RevitAPI, the functions you used in each SDK might vary. 

```cs
if (RIR.Revit.ActiveUIApplication.Application.VersionNumber == 2019)
{
    // do stuff that is related to Revit 2019
}
else
{
    // do other stuff
}
```
