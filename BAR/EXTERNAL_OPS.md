This tutorial is designed to guide developers through the options, processes and motivations for adding Ops outside the core [imagej-ops](https://github.com/imagej/imagej-ops) project.

# Make your first Op

At the most fundamental level, an Op is a SciJava [Plugin](https://github.com/scijava/scijava-common/blob/scijava-common-2.47.0/src/main/java/org/scijava/plugin/Plugin.java) encapsulating a piece of functionality, which can be discovered and used by a central [OpService](https://github.com/imagej/imagej-ops/blob/imagej-ops-0.18.0/src/main/java/net/imagej/ops/OpService.java).

At a minimum, creating an Op requires two pieces - an interface, and an implementation

## Create your Interface ##

```java
package bar;

import net.imagej.ops.Op;

/**
 * Op interface for calculating the greatest common divisor (GCD)
 */
public interface GCD extends Op {
  // Ops can be called by name, defined as properties of the Op interface
  String NAME = "gcd";
  // An alias is OPTIONAL but gives users additional ways to call your Op.
  // For example - the GCD is also called greatest common factor (GCF).
  String ALIASES = "gcf";
}
```

## Implement your Op ##

```java
package bar;

import org.scijava.ItemIO;
import org.scijava.plugin.Attr;
import org.scijava.plugin.Parameter;
import org.scijava.plugin.Plugin;

// The Plugin annotation allows this Op to be discovered by the OpService.
// We declare the type of op, the name of the op, and any optional aliases.
@Plugin(type = GCD.class, name = GCD.NAME, attrs = { @Attr(name = "aliases", value = GCD.ALIASES) })
public class DefaultGCD implements GCD {

  // -- Inputs --

  // We want our GCD function to have two inputs. These are declared using @Parameter notation
  @Parameter
  private double a;

  @Parameter
  private double b;

  // -- Outputs --

  // This op will return a single value (the computed GCD) so we declare an output parameter

  @Parameter(type = ItemIO.OUTPUT)
  private double result;

  @Override
  public void run() {
    // The job of the run method is to populate any outputs using the inputs
    result = computeGCD(a, b);
  }

  private double computeGCD(final double p1, final double p2) {
    return p2 == 0 ? p1 : computeGCD(p2, p1%p2);
  }
}
```

## Use your Op ##

With these two components, you can start using your Op - for example, in the [script editor](http://imagej.net/Scripting):

```python
# @OpService ops

from bar import GCD

# We look up our Op by asking the framework to find an implementation of the
# GCD interface that could handle these inputs
gcd = ops.op(GCD, 7, 10);

# Print the Op instance, to verify our Op is picked up
print(gcd)

# The OpService can print useful information about any Op..
print(ops.info(gcd))

# .. including its usage
print(ops.help("gcd"))

# We can also run an Op by passing the name or alias we defined
# Print the result of running our Op
print(ops.run("gcf", 20, 7))

```

# Group your Ops in a Namespace

Calling our Ops by name is not type-safe, and importing each interface is tedious. If you are going to provide a collection of Ops, a better way to package them is within a custom [Namespace](https://github.com/imagej/imagej-ops/blob/imagej-ops-0.18.0/src/main/java/net/imagej/ops/Namespace.java).

## Create your Interface(s) ##

```java
package bar;

import org.scijava.plugin.Plugin;

import net.imagej.ops.AbstractNamespace;
import net.imagej.ops.Namespace;
import net.imagej.ops.Op;
import net.imagej.ops.OpMethod;

@Plugin(type = Namespace.class)
public class BAR extends AbstractNamespace {

  @Override
  public String getName() {
    return "name";
  }

  // -- BAR Namespace Op interfaces --

  // We can make all of our interfaces nested classes.
  // This allows references to take the form of "Namespace.Op" which
  // can make things easier to understand.

  public interface GCD extends Op {
    // Note that the name and aliases are prepended with Namespace.getName
    String NAME = "bar.gcd";
    String ALIASES = "bar.gcf";
  }

  // -- BAR Namespace built-in methods --

  // Built-in methods provide type-safe methods for accessing Ops
  // in a namespace.

  // We always provide an Object... constructor that can be passed directly to the
  // OpService.run method

  @OpMethod(op = bar.BAR.GCD.class)
  public Object gcd(final Object... args) {
    return ops().run(bar.BAR.GCD.class, args);
  }

  // But we can also type-narrow our inputs and returns with our knowledge of the Op
  // implementations

  @OpMethod(op = bar.BAR.GCD.class)
  public double gcd(final double a, final double b) {
    return (Double) ops().run(bar.BAR.GCD.class, a, b);
  }
}

```

## Implement your Op(s) ##

The implementation is essentially the same as with single Ops, although we do have to update our class references.

Since the implementations are not accessed directly typically, whether they are grouped as nested classes or provided individually is less important than for the interfaces.

```java
package bar;

import org.scijava.ItemIO;
import org.scijava.plugin.Attr;
import org.scijava.plugin.Parameter;
import org.scijava.plugin.Plugin;

@Plugin(type = BAR.GCD.class, name = BAR.GCD.NAME, attrs = { @Attr(name = "aliases", value = BAR.GCD.ALIASES) })
public class DefaultGCD implements BAR.GCD {

  // -- Inputs --

  @Parameter
  private double a;

  @Parameter
  private double b;

  // -- Outputs --

  @Parameter(type = ItemIO.OUTPUT)
  private double result;

  @Override
  public void run() {
    result = computeGCD(a, b);
  }

  private double computeGCD(final double p1, final double p2) {
    return p2 == 0 ? p1 : computeGCD(p2, p1%p2);
  }
}

```

## Use your Op(s) ##

We can still use our Op through the OpService:

```python
# @OpService ops

print(ops.run("bar.gcd", 20, 15))
```

But we can also use our built-in methods:

```python
# @bar.BAR bar

print(bar.gcd(20, 15))
```

This is especially useful in environments with code completion.

Namespaces also present an easy way for users to find information about available functionality using the base Help ops:

```python
# @OpService ops
# @bar.BAR bar

# Print usage for all ops in the BAR namespace
print(ops.help(bar))
```

# Potential next steps

## Create a helper service for your Namespace

SciJava [Services](https://github.com/scijava/scijava-common/blob/scijava-common-2.47.0/src/main/java/org/scijava/service/Service.java) are a general workhorse in a given SciJava context. There is a single instance of each Service created per context, so they are a common container for static utility style methods.

When developing an external Namespace, an immediate benefit to creating a corresponding Service is that it provides an initialization hook for registering your new Namespace outside the Ops framework - in particular, with the SCriptService:

```java

package bar;

import org.scijava.plugin.Parameter;
import org.scijava.plugin.Plugin;
import org.scijava.script.ScriptService;
import org.scijava.service.AbstractService;
import org.scijava.service.Service;

import net.imagej.ImageJService;

@Plugin(type = Service.class)
public class BARService extends AbstractService implements ImageJService {

  @Parameter
  private ScriptService scriptService;

  @Override
  public void initialize() {
    // Register this namespace with the ScriptService so we can drop package prefixes
    // in script parameters, allowing:
    // @BAR
    // instead of
    // @bar.BAR
    scriptService.addAlias(bar.BAR.class);
  }
}
```
Now we can drop package prefixes when using our Namespace in scripts:

```python
# @OpService ops
# @BAR bar

# Print usage for all ops in the BAR namespace
print(ops.help(bar))
```

## Distribute scripts demonstrating how your Ops should be used

The ImageJ [script editor](http://imagej.net/Scripting) automatically collects scripts located in <code>src/main/resources/script_templates</code>. For example, if we create a file:

<code>src/main/resources/script_templates/BAR/GCD.py</code>

with contents:

```python
# @BAR bar
# @float a
# @float b

print("Greatest common divisor of " + str(a) + " and " + str(b) + " is: " + str(bar.gcd(a, b)))
```

Then users will be able to select <code>Templates > BAR > GCD</code> from the script editor window, to automatically load the script and select the correct script language (python in this case).

This is an easy way to provide a starting point for development using your Ops.

## Create additional implementations for your Ops interfaces
### Specialize for varied input types

## Auto-generate your Op implementations using templates

## Write unit-tests to ensure coverage of built-in methods for your Ops




| [Home] | [Analysis] | [Data Analysis] | [Annotation] | [Segmentation] | [Tools] | [Plugins][Java Classes] | [lib] | [Snippets] | [IJ] |
|:------:|:----------:|:---------------:|:------------:|:--------------:|:-------:|:-----------------------:|:-----:|:----------:|:----:|

[Home]: https://github.com/tferr/Scripts#ij-bar
[Analysis]: https://github.com/tferr/Scripts/tree/master/BAR/src/main/resources/scripts/BAR/Analysis/Analysis#analysis
[Annotation]: https://github.com/tferr/Scripts/tree/master/BAR/src/main/resources/scripts/BAR/Annotation/Annotation#annotation
[Data Analysis]: https://github.com/tferr/Scripts/tree/master/BAR/src/main/resources/scripts/BAR/Data_Analysis#data-analysis
[Segmentation]: https://github.com/tferr/Scripts/tree/master/BAR/src/main/resources/scripts/BAR/Segmentation/Segmentation#segmentation
[Tools]: https://github.com/tferr/Scripts/tree/master/Tools#tools-and-toolsets
[Java Classes]: https://github.com/tferr/Scripts/tree/master/BAR#java-classes
[lib]: https://github.com/tferr/Scripts/tree/master/lib#lib
[Snippets]: https://github.com/tferr/Scripts/tree/master/Snippets#snippets
[IJ]: http://imagej.net/BAR
