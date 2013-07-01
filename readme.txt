This extension enables Bindings to declare that they are a Scope for other objects created 'beneath' them in Resolution hierarchy. Dependent Services can now use the relevant object as their Scope.

Additionally, the extension adds
- InCallScope() which maintains a single instance for all objects created within a given Kernel.Get() call
- InParentScope() which means an object has the same lifecycle as the object it is injected into.

InNamedScope
============
Example: You have an ExcelSheet that is shared between two components, one to draw the sheet and one to update the calculated values on each cell. These two aspects of the sheet management functionality have a shared dependency on SheetDataRepository and the same instance needs to be used between them. You cant use Singleton Scope because you want to be able to create multiple sheets. In this case the Bindings can be defined like this:

  const string SheetScopeName = "ExcelSheet";
  Bind<ExcelSheet>().ToSelf().DefinesNamedScope(SheetScopeName);
  Bind<SheetPresenter>().ToSelf();
  Bind<SheetCalculator>().ToSelf();
  Bind<SheetDataRepository>().ToSelf().InNamedScope(SheetScopeName);

In conjunction with Ninject.Extensions.ContextPreservation, this Scope Type can also be used for Services that are not Resolved as direct dependency within the same Resolution Request, but are instead Resolved some time later, e.g. by a factory that was created as dependency as part of an initial Composition. Imagine in the above scenario that the SheetCalculator is supplied via Constructor Injection with a CellCalculatorFactor which has a CreateCellCalculator() method that is used by the Sheet calculator to create a CellCalculator whenever a formula is added to a cell. The cell calculator now needs a SheetDataRepository...

  this.kernel.Load(new NamedScopeModule());
  this.kernel.Load(new ContextPreservationModule());

  const string SheetScopeName = "ExcelSheet";
  Bind<ExcelSheet>().ToSelf().DefinesNamedScope(SheetScopeName);
  Bind<SheetPresenter>().ToSelf();
  Bind<SheetCalculator>().ToSelf();
  Bind<CellCalculatorFactory>().ToSelf();
  Bind<CellCalculator>().ToSelf();
  Bind<SheetDataRepository>().ToSelf().InNamedScope(SheetScopeName);

NB: Without the ContextPreservationModule being loaded, you will get an UnknownScopeException.

InCallScope
===========
If this type of scope is used for a Binding, only one instance is created per onecall to kernel.Get<IX>(). In the above scenario the first example can be written as:

  Bind<ExcelSheet>().ToSelf();
  Bind<SheetPresenter>().ToSelf();
  Bind<SheetCalculator>().ToSelf();
  Bind<SheetDataRepository>().ToSelf().InCallScope();
  
However the second scenario is not possible because the CreateCellCalculator requests on the factory would produce an individual new data repository for each Request.

There is one way in which InCallScope can be transparently passed through a Get when the Context Preservation Extension is used. Imagine that in the above scenario we have two different views onto the data repository using two interfaces, ICellValues and ICellFormulas. In this case the Bindings can be done as follows:

  Bind<ExcelSheet>().ToSelf();
  Bind<SheetPresenter>().ToSelf();
  Bind<SheetCalculator>().ToSelf();
  Bind<SheetDataRepository>().ToSelf().InCallScope();
  Bind<ICellValues>().ToMethod(ctx => ctx.ContextPreservingGet<SheetDataRepository>());  
  Bind<ICellFormulas>().ToMethod(ctx => ctx.ContextPreservingGet<SheetDataRepository>());  
  
Note that for the last two bindings, the BindInterfaceToBinding() extension enables the following more compact alternate definition:
  kernel.BindInterfaceToBinding<ICellValues, SheetDataRepository>();
  kernel.BindInterfaceToBinding<ICellFormulas, SheetDataRepository>();  
  
InParentScope
=============
This is basically the same as InTransientScope() at least when it comes to the lifecycle and the decision as to which object is injected. As with InTransientScope a new instance is injected for each dependency. The difference is that bindings with this scoping will be deactivated alongside the object that gets the instance injected into it as that gets collected by the garbage collector (or Release()d) i.e., the Kernel Dispose()s the object when it is not used anymore.

CreateNamedScope
================
The extension provides the functionallity to create a resolution root that is a named scope for all objects created by this resolution root. E.g.

kernel.Bind<IFoo>().To<Foo>().InNamedScope("TheScopeName");
kernel.Bind<IBar>().To<Bar>().InSingletonScope();

IResolutionRoot scope = kernel.CreateNamedScope("TheScopeName");
var foo = scope.Get<IFoo>();
var bar = scope.Get<IBar>();

In this case foo will live until scope is Disposed, but bar will live until the Kernel is Disposed. This feature should usually be used for integration with frameworks that need to create several instances that need to be Disposed together. E.g. ASP.NET MVC or NServiceBus integration. For applications, the earlier approaches above should be preferred.
