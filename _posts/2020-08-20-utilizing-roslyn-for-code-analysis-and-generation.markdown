---
layout: post
title:  "Utilizing Roslyn for Code Analysis and Generation"
date:   2020-08-20 12:05:34 +0100
categories: C# .NET 
---
This is a repost of a blog post I wrote for my [company's blog](https://klika.us/blog/utilizing-roslyn-for-code-analysis-and-generation) in 2020. They have some interesting topics, give them a read.

Want to improve the security of your code or increase your coding speed?

Using Roslyn, the .NET compiler platform, both and more can be achieved.

The package Microsoft.CodeAnalysis and related packages provide the tools to create your own code analysis packages or to create custom boilerplate code generation.

As for code analysis there already exist many great packages that you get as an ordinary NuGet Packages.

I'll demonstrate bellow what kinds of analyzers exist, what they provide and also how you can generate C# code by using Roslyn.

# Code Analysis

By using Visual Studio to write C# code, by default, you have already been using Roslyn Code Analyzers. The default ones provide basic code security and typo correction.

You can add more powerful code analyzers that provide more advanced code security and code formatting options or even write your own ones if necessary. 

You can find all the analyzers that a project uses under the Dependencies of a single project.

![dependencies](/images/2020-08-20/Untitled1.png)

Existing analyzers are added to the project as NuGet packages. For example I'll add the Roslynator analyzer trough the Package Manager Cosole.

```csharp
Install-Package Roslynator.Analyzers -Version 2.3.0
```

Now, looking under the Analyzers you will see the installed Roslynator analyzers. Under Roslynator.CSharp.Analyzers are all the rules that come with the package.

![rules](/images/2020-08-20/Untitled2.png)

Right clicking a rule you can edit its severity rule — setting a rule as an error will prevent compilation if it is present.

![edit](/images/2020-08-20/Untitled3.png)

![edit2](/images/2020-08-20/Untitled4.png)

This way you can enforce a set of rules for you code every time you build it.

Though these analyzers are not as powerful as NDepend, ReSharper or other proprietary code analysis tools, they are free and easy to set up.

Some popular analyzers are:

- FxCop
- StyleCop
- SecurityCodeScan
- Roslynator

# Code Generation

Another great feature of Roslyn is generating C# code. 
This can be achieved by using the SyntaxGenerator class from the Microsoft.CodeAnalysis.Editing package.

You can assemble an entire class by generating parts like Namespace Imports, Fields, Properties, Constructors, Methods as instances of SyntaxNode and then creating a single Compilation Unit from those parts.

This can be used to generate classes for the underlying layers of an application. Like service an layer class with same CRUD method implementation but for different models.
The same is achieved when using the Scaffold option in ASP.NET to generate controllers with boilerplate code.  
Creating your own boilerplate can greatly increase your coding speed and increase consistency across the codebase.

Bellow is an small example of how this is done by generating a class with a single field, constructor and a method for the Example model class.

```csharp
class Program
    {
        static void Main(string[] args)
        {
            var code = @"
            namespace Classes
            {
                class Example
                {
                    public int Id { get; set; }
                }
            }";
            Console.WriteLine(Generate(code));
            
        }

        public static string Generate(string code)
        {
            var node = CSharpSyntaxTree.ParseText(code).GetRoot();
            var classNode = node.DescendantNodes().OfType<ClassDeclarationSyntax>().FirstOrDefault();
            var modelClassName = classNode.Identifier.Text;

            AdhocWorkspace adhocWorkspace = new AdhocWorkspace();
            var syntaxGenerator = SyntaxGenerator.GetGenerator(adhocWorkspace, LanguageNames.CSharp);

            // using
            var usingDeclarationSystem = syntaxGenerator.NamespaceImportDeclaration("System");
            var usingDeclarationSystemLinq = syntaxGenerator.NamespaceImportDeclaration("System.Linq");
            var usingDeclarationSystemCollectionsGeneric = syntaxGenerator.NamespaceImportDeclaration("System.Collections.Generic");
            var usingDeclarationSystemThreadingTasks = syntaxGenerator.NamespaceImportDeclaration("System.Threading.Tasks");

            // _dbContext
            var appDbContextField = syntaxGenerator.FieldDeclaration("_dbContext", SyntaxFactory.ParseTypeName("AppDbContext"), Accessibility.Private, DeclarationModifiers.ReadOnly);

            var constructorParameters = new[] { syntaxGenerator.ParameterDeclaration("appDbContext", SyntaxFactory.ParseTypeName("AppDbContext")) };
            var constructorBody = new[] {
                syntaxGenerator.AssignmentStatement(
                    syntaxGenerator.IdentifierName("_dbContext"),
                    syntaxGenerator.IdentifierName("appDbContext"))};
            // ctor
            var constructor = syntaxGenerator.ConstructorDeclaration($"{modelClassName}Services", constructorParameters, Accessibility.Public,
                statements: constructorBody);

            
            var methodBody = new List<SyntaxNode> { SyntaxFactory.ParseStatement($"return await _dbContext.{modelClassName}s.ToListAsync();") };
            
            // GetAsync()
            var getAsyncMethod = syntaxGenerator.MethodDeclaration("GetAsync", null, null,
                SyntaxFactory.ParseTypeName($"Task<IEnumerable<{modelClassName}>>"),
                Accessibility.Public,
                DeclarationModifiers.Async, methodBody);

            var members = new[]
            {
                appDbContextField,
                constructor,
                getAsyncMethod
            };

            // Class
            var classDefinition = syntaxGenerator.ClassDeclaration(
                name:$"{modelClassName}Services", 
                typeParameters: null, 
                accessibility: Accessibility.Public, 
                modifiers: DeclarationModifiers.None,
                baseType: null,
                members: members);

            // Namespaces
            var namespaceDeclaration = syntaxGenerator.NamespaceDeclaration("Services", classDefinition);

            // Compilation unit
            return syntaxGenerator.CompilationUnit(
                    usingDeclarationSystem,
                    usingDeclarationSystemLinq,
                    usingDeclarationSystemCollectionsGeneric,
                    usingDeclarationSystemThreadingTasks, 
                    namespaceDeclaration)
                .NormalizeWhitespace()
                .ToFullString();
        }
    }
```

Executing the code above, in a console application, returns the following:

```csharp
using System;
using System.Linq;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace Services
{
    public class ExampleServices
    {
        private readonly AppDbContext _dbContext;
        public ExampleServices(AppDbContext appDbContext)
        {
            _dbContext = (appDbContext);
        }

        public async Task<IEnumerable<Example>> GetAsync()
        {
            return await _dbContext.Examples.ToListAsync();
        }
    }
}
```

The example shown above is just a lightweight presentation how Roslyn can be utilized. 
A more complex boilerplate generator, with custom search, create and update methods can be achieved by using attributes on your model properties to denote which of the operations can be done on which property.

From the above example you can just add attributes in plain text above the property however if you are reading the model class from a project, like shown in the snippet bellow, then you can just add attributes to the project. They won't have any implementation as only their name is important.

```csharp
var code = new StreamReader("..\\..\\..\\Classes\\Person.cs").ReadToEnd();
```

Then get the properties with their attributes.

```csharp
IEnumerable<MemberDeclarationSyntax> members = classNode.DescendantNodes().OfType<MemberDeclarationSyntax>();
List<KeyValuePair<string, string>> propertiesWithAttributes = new List<KeyValuePair<string, string>>();
foreach (var memberDeclarationSyntax in members)
            {
                var attributeName = new List<string>();
                var property = memberDeclarationSyntax as PropertyDeclarationSyntax;
                var attributes = property.AttributeLists.ToList();
                foreach (var attributeListSyntax in attributes)
                {
                    attributeName.Add(attributeListSyntax.Attributes.First().Name.NormalizeWhitespace().ToFullString());
                }

                if (attributeName != null)
                    attributeName.ForEach(x => propertiesWithAttributes.Add(new KeyValuePair<string, string>(property.Identifier.Text, x)));
            }
```

Afterwards you can dynamically generate SyntaxNode instances for search, create and update method bodies with whatever custom logic you need.

Also you can generate method, constructor, getter and setter bodies more programmatically instead of hardcoding the lines of code, as I did.  

```csharp
var numSyntaxNode = new List<SyntaxNode>
                { syntaxGenerator.AssignmentStatement(
                    left:syntaxGenerator.IdentifierName("var num"),
                    right:syntaxGenerator.IdentifierName("1"))};
```

# Conclusion?

The Roslyn platform can be a powerful tool that aids during software development. Taking some time to learn to utilize it can result in a more consistent code base and quicker develop time.