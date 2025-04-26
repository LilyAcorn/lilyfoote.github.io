+++
title = "Quirks in Django's template language"
date = 2025-04-26
draft = false
categories = ["Django"]
tags = ["django", "templates"]
+++

## Django Rusty Templates

Since September 2024, I have been building a reimplementation of Django's templating language in Rust, called [Django Rusty Templates](https://github.com/LilyFoote/django-rusty-templates). During this process, I have discovered several weird edge-cases and bugs.

### Scientific notation

Django supports writing floats in scientific notation (e.g. `6.28e23`) in templates. However using a negative exponent was not supported:

{{< narrow >}}
```python
"{{ foo|default:5.2e-3 }}"

Traceback (most recent call last):
  TemplateSyntaxError:
    Could not parse the remainder:
      '-3' from 'foo|default:5.2e-3'
```
{{< /narrow >}}
{{< wide >}}
```python
>>> from django.template.base import Template
>>> Template("{{ foo|default:5.2e-3 }}")
Traceback (most recent call last):
  ...
    raise TemplateSyntaxError(
django.template.exceptions.TemplateSyntaxError:
    Could not parse the remainder: '-3' from 'foo|default:5.2e-3'
```
{{< /wide >}}

This was caused by a bug in the regular expression Django uses to identify variables and numbers.

I reported this in [ticket #35816](https://code.djangoproject.com/ticket/35816) and [a fix has been merged to `main`](https://github.com/django/django/pull/19213).

### The upper filter is not HTML-safe

When implementing the `lower` filter, I noticed that it is marked as HTML-safe, so after the filter is applied no more HTML escaping is required. However, the `upper` filter is marked as HTML-unsafe, leading to any HTML content being escaped in the rendered template:

{{< narrow >}}
```python
>>> from django.template \
    import engines
>>> from django.utils.safestring \
    import mark_safe

>>> template = "{{ content|upper }}"
>>> template = engines["django"]
        .from_string(template)
>>> content = mark_safe(
        "<p>Hello World!</p>")

>>> template.render(
    {"content": content})
'&lt;P&gt;HELLO WORLD!&lt;/P&gt;'
```
{{< /narrow >}}
{{< wide >}}
```python
>>> from django.template import engines
>>> from django.utils.safestring import mark_safe

>>> template = "{{ content|upper }}"
>>> template = engines["django"].from_string(template)
>>> content = mark_safe("<p>Hello World!</p>")

>>> template.render({"content": content})
'&lt;P&gt;HELLO WORLD!&lt;/P&gt;'
```
{{< /wide >}}

I reported this in [ticket #36049](https://code.djangoproject.com/ticket/36049), but it turns out this is more subtle than I expected. In code review, [it was pointed out that this is tested behaviour](https://github.com/django/django/pull/18988#discussion_r1904285919) and the test explains:

> The "upper" filter messes up entities (which are case-sensitive), so it's not safe for non-escaping purposes.

This means that using `upper` on a valid HTML entity like `&amp;` would convert it to `&AMP;` which is invalid. This is further complicated by the existence of case-sensitive entities like `&Aacute;` and `&aacute` (`Á` and `á`), which we might want to handle carefully in a fully HTML-aware `upper` or `lower` filter.

Given all these subtleties, I recommend avoiding using the `lower`, `title` and `upper` filters with any content that may contain HTML.

### Exception swallowing in {% if %} tags

Django's `{% if %}` template tag supports several operators (`and`, `or`, `not`, `==`, etc) which are evaluated to a truthy or falsey result. This is straightforward as long as no exceptions are raised. When an exception is raised, these operators catch the exception and return `False` instead.

Unfortunately, the exception is sometimes caught late, leading to counter-intuitive results. For example:

{{< narrow >}}
```django
{% if missing|default:missing
    or True %}
```
{{< /narrow >}}
{{< wide >}}
```django
{% if missing|default:missing or True %}
```
{{< /wide >}}

I would expect this to always evaluate as truthy. However, when the variable `missing` is not in the context, the `default` filter will raise a `VariableDoesNotExist`. The `or` operator will catch this exception and evaluate to `False`.

[Ticket 17664](https://code.djangoproject.com/ticket/17664) has a lot of discussion about this issue.

### Conclusion

Reimplementing an existing system is an excellent way to discover edge-cases and bugs. While Django is a very robust project, it still has a few rough edges it can be helpful to be aware about.
