+++
title = "Sharing a mutable reference with Python"
date = 2025-09-02
draft = false
categories = ["Rust", "Python", "PyO3"]
tags = ["django", "templates", "rust", "python", "pyo3"]
+++

## Background

As part of [my ongoing project](https://github.com/LilyAcorn/django-rusty-templates) to reimplement Django's templating language in Rust, I have been adding support for [custom template tags](https://docs.djangoproject.com/en/5.2/howto/custom-template-tags/#writing-custom-template-tags).

### Simple tags
The simplest custom tag will look something like:

{{< narrow >}}
```python
# time_tags.py
from datetime import datetime
from django import template

register = template.Library()


@register.simple_tag
def time(format_string):
  now = datetime.now()
  return now.strftime(format_string)
```
{{< /narrow >}}
{{< wide >}}
```python
# time_tags.py
from datetime import datetime
from django import template

register = template.Library()


@register.simple_tag
def current_time(format_string):
    return datetime.now().strftime(format_string)
```
{{< /wide >}}

This can then be used in a Django template like:

{{< narrow >}}
```html
{% load time from time_tags %}
<p>Time: {% time '%H:%M:%S' %}</p>
```
{{< /narrow >}}
{{< wide >}}
```html
{% load current_time from time_tags %}
<p>The time is {% current_time '%H:%M:%S' %}</p>
```
{{< /wide >}}

### The context

Django's templating language uses an object called a `context` to provide dynamic data to the template renderer. This mostly behaves like a Python dictionary.

{{< details summary="details" >}}Technically, Django's context contains a list of dictionaries. This allows for temporarily changing the value of a variable, for example within a `{% for %}` loop, while keeping the old value for later use.{{< /details >}}

A simple tag can be defined that takes the `context` as the first variable:

{{< narrow >}}
```python
# time_tags.py
from datetime import datetime
from django import template

register = template.Library()


@register.simple_tag(
    takes_context=True)
def time(context, format_string):
  timezone = context["timezone"]
  now = datetime.now(tz=timezone)
  return now.strftime(format_string)
```
{{< /narrow >}}
{{< wide >}}
```python
# time_tags.py
from datetime import datetime
from django import template

register = template.Library()


@register.simple_tag(takes_context=True)
def current_time(context, format_string):
    timezone = context["timezone"]
    return datetime.now(tz=timezone).strftime(format_string)
```
{{< /wide >}}

## Django Rusty Templates

In Django Rusty Templates, I have defined a `context` as a Rust `struct`. Here's a simplified version:

{{< narrow >}}
```rust
pub struct Context {
  context: HashMap<
    String, Py<PyAny>>,
}
```
{{< /narrow >}}
{{< wide >}}
```rust
pub struct Context {
    context: HashMap<String, Py<PyAny>>,
}
```
{{< /wide >}}

When rendering a template tag, the context is passed to the `render` method as a mutable reference:

{{< narrow >}}
```rust
trait Render {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult;
}
```
{{< /narrow >}}
{{< wide >}}
```rust
trait Render {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult;
}
```
{{< /wide >}}

This is natural when working purely in Rust but custom template tags require passing the context to Python, which doesn't understand Rust lifetimes.

The standard way of connecting Python and Rust is with [PyO3](https://pyo3.rs/). To pass a Rust type to Python, we can wrap it in a [`#[pyclass]`](https://pyo3.rs/v0.26.0/class.html):

{{< narrow >}}
```rust
#[pyclass]
struct PyContext {
  context: Context,
}

#[pymethods]
impl PyContext {
  // Methods for Python to read from
  // the context
}
```
{{< /narrow >}}
{{< wide >}}
```rust
#[pyclass]
struct PyContext {
    context: Context,
}

#[pymethods]
impl PyContext {
    // Methods for Python to read from the context
}
```
{{< /wide >}}

And then we can create this in our `render` method:

{{< narrow >}}
```rust
struct CustomTag {
  func: Py<PyAny>,
  takes_context: bool,
}

impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let py_context =
        PyContext { context };
      let content = self.func
        .bind(py)
        .call1((py_context,))?;
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
struct CustomTag {
    func: Py<PyAny>,
    takes_context: bool,
}

impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let py_context = PyContext { context };
            let content = self.func.bind(py).call1((py_context,))?;
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

Unfortunately, this doesn't compile because `PyContext` requires an owned value, not a mutable reference:

{{< narrow >}}
```rust
error[E0308]: mismatched types
   --> src/render/tags.rs:807:40
    |
807 | let py_context =
        PyContext { context };
    |               ^^^^^^^
            expected `Context`,
            found `&mut Context`
```
{{< /narrow >}}
{{< wide >}}
```rust
error[E0308]: mismatched types
   --> src/render/tags.rs:807:40
    |
807 |             let py_context = PyContext { context };
    |                                          ^^^^^^^ expected `Context`,
                                                       found `&mut Context`
```
{{< /wide >}}

## Turning a mutable reference into an owned value

To make progress, we need to find a way to get an owned version of `context`. To do this, I turned to [`std::mem::take`](https://doc.rust-lang.org/std/mem/fn.take.html), which replaces the data pointed to by `&mut context` with an empty value (via [the `Default` trait](https://doc.rust-lang.org/std/default/trait.Default.html)) and returns an owned value:

{{< narrow >}}
```rust
#[derive(Default)]
pub struct Context {
  context: HashMap<
    String, Py<PyAny>>,
}

impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let context =
        std::mem::take(context);
      let py_context =
        PyContext { context };
      let content = self.func
        .call1(py, (py_context,))?;
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
#[derive(Default)]
pub struct Context {
  context: HashMap<String, Py<PyAny>>,
}

impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let context = std::mem::take(context);
            let py_context = PyContext { context };
            let content = self.func.call1(py, (py_context,))?;
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

## Moving the owned value back into the mutable reference

This works very well for giving Python access to the `context`. However, once the custom tag's rendering logic has run we need to regain ownership of the `context` for use in other Rust tags. To do this, we turn to [`std::mem::replace`](https://doc.rust-lang.org/std/mem/fn.replace.html):

{{< narrow >}}
```rust
impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let swapped_context =
        std::mem::take(
          swapped_context);
      let py_context = PyContext {
          context: swapped_context
      };
      let content = self.func
        .call1(py, (py_context,))?;
      let _ = std::mem::replace(
        context,
        py_context.context);
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let swapped_context = std::mem::take(context);
            let py_context = PyContext { context: swapped_context };
            let content = self.func.call1(py, (py_context,))?;
            let _ = std::mem::replace(context, py_context.context);
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

Unfortunately, this again does not compile:

{{< narrow >}}
```rust
error[E0382]: use of moved value:
                `py_context.context`
   --> src/render/tags.rs:815:48
    |
813 | let py_context = PyContext { 
        context: swapped_context };
    |     ----------
          move occurs because
          `py_context` has type
          `PyContext`, which does
          not implement the `Copy`
          trait
814 | let content = self.func.call1(
        py, (py_context,))?;
    |        ----------
             value moved here
815 | let _ = std::mem::replace(
        context,
        py_context.context);
    |   ^^^^^^^^^^^^^^^^^^
        value used here after move
```
{{< /narrow >}}
{{< wide >}}
```rust
error[E0382]: use of moved value: `py_context.context`
   --> src/render/tags.rs:815:48
    |
813 |             let py_context = PyContext { context: swapped_context };
    |                 ---------- move occurs because `py_context` has type
                                    `PyContext`, which does not implement
                                    the `Copy` trait
814 |             let content = self.func.call1(py, (py_context,))?;
    |                                                ---------- value moved here
815 |             let _ = std::mem::replace(context, py_context.context);
    |                                                ^^^^^^^^^^^^^^^^^^
                                                        value used here after move
```
{{< /wide >}}

To get around this, we can use an [`Arc` (atomic reference count)](https://doc.rust-lang.org/std/sync/struct.Arc.html) to send Python a clone of `py_context` rather than moving it out of scope. We can also remove the `context` from the `Arc` with [`Arc::try_unwrap`](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.try_unwrap):

{{< narrow >}}
```rust
#[pyclass]
#[derive(Clone)]
struct PyContext {
  context: Arc<Context>,
}

impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let swapped_context =
        std::mem::take(
          swapped_context).into();
      let py_context = PyContext {
        context: swapped_context
      };
      let content = self.func.call1(
        py, (py_context.clone(),)?;
      let inner_context = match 
          Arc::try_unwrap(
            py_context.context) {
        Ok(inner_context) => {
          inner_context
        }
        Err(_) => todo!(),
      };
      let _ = std::mem::replace(
        context, inner_context);
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
#[pyclass]
#[derive(Clone)]
struct PyContext {
    context: Arc<Context>,
}

impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let swapped_context = std::mem::take(context);
            let py_context = PyContext { context: swapped_context };
            let content = self.func.call1(py, (py_context.clone(),))?;
            let inner_context = match Arc::try_unwrap(py_context.context) {
                Ok(inner_context) => inner_context,
                Err(_) => todo!(),
            };
            let _ = std::mem::replace(context, inner_context);
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

This works great when Python doesn't keep a reference to the `PyContext` object. This means the reference count of the `Arc` is one and `Arc::try_unwrap` will succeed. If the custom tag implementation keeps a reference around for some reason, we cannot take ownership. Instead we must fall back to cloning the inner context:

{{< narrow >}}
```rust
#[derive(Default)]
pub struct Context {
  context: HashMap<
    String, Py<PyAny>>,
}

impl Context {
  fn clone_ref(
      &self, py: Python<'_>
  ) -> Self {
    Self {
      context: self
        .context
        .iter()
        .map(|(k, v)| (
          k.clone(),
          v.clone_ref(py),
        ))
        .collect(),
    }
  }
}

impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let swapped_context =
        std::mem::take(
          swapped_context).into();
      let py_context = PyContext {
        context: swapped_context
      };
      let content = self.func.call1(
        py, (py_context.clone(),)?;
      let inner_context = match 
          Arc::try_unwrap(
            py_context.context) {
        Ok(inner_context) => {
          inner_context
        }
        Err(inner_context) => {
          inner_context
            .clone_ref(py)
        }
      };
      let _ = std::mem::replace(
        context, inner_context);
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
#[derive(Default)]
pub struct Context {
    context: HashMap<String, Py<PyAny>>,
}

impl Context {
    fn clone_ref(&self, py: Python<'_>) -> Self {
        Self {
            context: self
                .context
                .iter()
                .map(|(k, v)| (k.clone(), v.clone_ref(py)))
                .collect(),
        }
    }
}

impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let swapped_context = std::mem::take(context);
            let py_context = PyContext { context: swapped_context };
            let content = self.func.call1(py, (py_context.clone(),))?;
            let inner_context = match Arc::try_unwrap(py_context.context) {
                Ok(inner_context) => inner_context,
                Err(inner_context) => inner_context.clone_ref(py),
            };
            let _ = std::mem::replace(context, inner_context);
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

Note that we need to use [the `clone_ref` method](https://docs.rs/pyo3/latest/pyo3/struct.Py.html#method.clone_ref) instead of `clone` because this handles Python's reference counts correctly.

## Mutating the context from Python

This is sufficient to grant Python read-only access to the `context`, but the `context` is designed to be mutated. To enable this, we need to protect the `context` from being mutably accessed from multiple threads. To do this, we can use a [`Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html), along with [PyO3's `MutexExt` trait](https://docs.rs/pyo3/latest/pyo3/sync/trait.MutexExt.html) which provides the `lock_py_attached` method to avoid deadlocking with the Python interpreter:

{{< narrow >}}
```rust
use pyo3::sync::MutexExt;

#[pyclass]
#[derive(Clone)]
struct PyContext {
  context: Arc<Mutex<Context>>,
}

impl PyContext {
  fn new(context: Context) -> Self {
    Self {
      context: Arc::new(
        Mutex::new(context)),
    }
  }
}

impl Render for CustomTag {
  fn render(
    &self,
    py: Python<'_>,
    context: &mut Context,
  ) -> RenderResult {
    if self.takes_context {
      let swapped_context =
        std::mem::take(
          swapped_context);
      let py_context =
        PyContext::new(
          swapped_context
      );
      let content = self.func.call1(
        py, (py_context.clone(),)?;
      let inner_context = match 
          Arc::try_unwrap(
            py_context.context) {
        Ok(inner_context) => {
          inner_context
            .into_inner().unwrap()
        }
        Err(inner_context) => {
          let guard = inner_context
            .lock_py_attached(py)
            .unwrap();
          guard.clone_ref(py)
        }
      };
      let _ = std::mem::replace(
        context, inner_context);
      Ok(content.to_string())
    }
  }
}
```
{{< /narrow >}}
{{< wide >}}
```rust
use pyo3::sync::MutexExt;

#[pyclass]
#[derive(Clone)]
struct PyContext {
    context: Arc<Mutex<Context>>,
}

impl Render for CustomTag {
    fn render(&self, py: Python<'_>, context: &mut Context) -> RenderResult {
        if self.takes_context {
            let swapped_context = std::mem::take(context);
            let py_context = PyContext { context: swapped_context };
            let content = self.func.call1(py, (py_context.clone(),))?;
            let inner_context = match Arc::try_unwrap(py_context.context) {
                Ok(inner_context) => inner_context
                    .into_inner
                    .expect(
                        "Mutex should be unlocked because Arc refcount is one."),
                Err(inner_context) => {
                    let guard = inner_context
                        .lock_py_attached(py)
                        .expect("Mutex should not be poisoned");
                    guard.clone_ref(py)
                }
            };
            let _ = std::mem::replace(context, inner_context);
            Ok(content.to_string())
        }
    }
}
```
{{< /wide >}}

## Conclusions

The inability of PyO3 to expose Rust structs that use lifetimes initially seems limiting, but PyO3 and Rust provide powerful tools to work around these limitations. [`std::mem::take`](https://doc.rust-lang.org/std/mem/fn.take.html), [`std::mem::replace`](https://doc.rust-lang.org/std/mem/fn.replace.html) and [`std::mem::swap`](https://doc.rust-lang.org/std/mem/fn.swap.html) allow for advanced manipulation of mutable references and owned values and [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) and [`Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html) are extremely useful for exposing shared mutable data to Python. PyO3's [`MutexExt`](https://docs.rs/pyo3/latest/pyo3/sync/trait.MutexExt.html) is essential for working with mutexes and Python together.

You can find the [full code implementing a simple custom tag here](https://github.com/LilyAcorn/django-rusty-templates/compare/3a309aa...9cbcfc6), with the extra details I omitted here for brevity and clarity.
