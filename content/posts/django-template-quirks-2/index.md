+++
title = "Quirks in Django's template language part 2"
date = 2025-07-20
draft = false
categories = ["Django", "Python"]
tags = ["django", "templates", "python"]
+++

## Django Rusty Templates

Since September 2024, I have been building a reimplementation of Django's templating language in Rust, called [Django Rusty Templates](https://github.com/LilyFoote/django-rusty-templates). During this process, I have discovered several weird edge-cases and bugs, [some of which I've talked about before](/posts/django-template-quirks/).

### Centring strings

Django supports a template filter for centring strings: `{{ "Django"|center:10 }}` which adds padding around the string to reach the provided length. In this simple case, the template is rendered as `··Django··` (where `·` represents a space character).

Things get weird when the total amount of padding required is an odd number. For example, `{{ "odd"|center:6 }}` becomes `·odd··`, but `{{ "even"|center:7 }}` becomes `··even·`. Note that the side with the extra space character is inconsistent.

The cause of this quirk is that Django's `center` filter directly calls Python's `str.center` method, [which maintains this behaviour for backwards compatibilty reasons](https://github.com/python/cpython/issues/67812).

It is possible to get more intuitive behaviour by using f-strings (or older format strings) with the `^` alignment option, which centres the string:

```python
>>> f"{'odd':^6}"
'·odd··'
>>> f"{'even':^7}"
'·even··'
```

But this isn't available in the Django template language without defining a custom filter or moving the centring logic out of the template.

Thanks to Moshe Nahmias for working on [implementing the `center` filter in Django Rusty Templates](https://github.com/LilyAcorn/django-rusty-templates/pull/91), and finding this quirk.
