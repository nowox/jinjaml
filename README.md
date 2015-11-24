# Jin<>JAML
An ambitious standalone templating engine for misc usages (C/C++, HTML, LaTeX, ...)

## What is Jin<>JAML?
JinJAML is a standalone command line tools (also available as module) written in [Python 2.7](https://www.python.org/download/releases/2.7/) allowing to populate templates with data issues from YAML files. 

This modules is based on [jinja2](http://jinja.pocoo.org/docs/dev/) and [HiYaPyCo](https://github.com/zerwes/hiyapyco). YAML files are validated using [pyKwalify](https://github.com/Grokzen/pykwalify). 

```
        .-------------------------.
        |                         v
      .---.                     .---.
      |YML| <----------.        |XSD| (Kwalify Syntax)
      '---'            |        '---'
config.default.yml     |   config.schema.yml
                       |                       .---------.
      .---. -----------'        .---.          |         |        .---.
      |YML| <-------------------|JML|--------->| JinJAML |------->| H |
      '---'                     '---'          |         |        '---'
    config.yml             template.h.jml      '---------'      template.h
```

Initial data are stored into [YAML](http://yaml.org/) files. Additionnaly to the `%YAML 1.2` specs, a YAML file can depend on other YAML files. These files can be used as external namespaces for tags replacement or as dependency. With the latter you can choose from two actions: merge or override. 

## A simple example

Here a very basic configuration:
```
# config.default.yml
%YAML 1.2
%TAG  !schema! config.schema.yml
---
foo: 42
bar: 
   - baz
   - qux
...
```

Which can be validated with a schema:
```
# config.default.yml
%YAML 1.2
--- !pykwalify
type: map
mapping: 
   foo:
      desc: Foo is a parameter used in this very configuration.
      type: int
      range: {min: 0; max: 42} 
   bar:
      desc: Bar can be used to list something useful.
      type: seq
      sequence: 
        type: str
        unique: True
...
```

A basic configuration can be written that inherit from the default configuration

```
# config.yml
%YAML 1.2
%TAG  !schema! config.schema.yml
---
<>: # JinJAML magic tag
   super: {file: config.default.yml; method: merge}
   
foo: 12
bar: 
   - quux
   - norf
other: !!int "{{foo}}"
...
```

Here is the template

```
{% datasource 'config.yml' %}
// Header file (other: {{other}})
#ifndef _FOO_H_
#define _FOO_H_
#define FOO_SIZE {{foo}}
{% for b in bar -%}
char *{{ b }};
{%- endfor %}
#endif // _FOO_H_
```

At the end, Jin<>JAML will process the template as follow: 

1. Read the template and check for `datasource`
2. Read the data source which is `config.yml` and extract the JinJAML `<>:` key
3. Validate with the schema `config.schema.yml` because it is linked to a schema
4. Check its dependance `config.default.yml`
5. Validate with the schema `config.schema.yml` because it is linked to a schema
6. Merge the two files together
7. Apply the inner template `{{foo}}`
8. Process the resulting YAML file into a dictionary

The resulting dictionary will be:

```
{'foo':12; 'bar':['baz','qux','quux','norf']; 'other': 12}
```

With this information, the template can be processed:

```
// Header file (other: 12)
#ifndef _FOO_H_
#define _FOO_H_
#define FOO_SIZE 12
char *baz;
char *qux;
char *quux;
char *norf;
#endif // _FOO_H_
```

## A bolder example: The expresso machine

Let's consider we are a company that manufacture expresso machines. Each expresso machine is based on the exact same electronic that comes in various models and each models have a particular configuration. 

The configuration is for example: 

- The available features
- The menu options
- The temperature and the pressure for each coffee brends
- The color code of the capsules

Each *firmware* will be build witht the enabled features for the concerned product. Menu, temperatures and other parameters are embedded into the C/C++ code base. The pressure and temperature are used for the documentation.

The complete configuration could be stored into a database such as sqlite or mongodb, however, it is eaiser here to store these files as plain text. The configuration can be under Git control and diff, compare and merge become very easy. 

So, we have here different files: 

```
# arch.yml
%YAML 1.2
%TAG  !schema! arch.schema.yml
---
brand: Exos
product_name: Expresso Maker
proc: ARM Cortex M0
...
```

```
# products.yml
%YAML 1.2
%TAG !schema! products.schema.yml
---
<>:
   import: {filename: arch.yml; as: arch}

products:   
   22: 
      name: {{ arch.brand }} Galaxy Coffee Maker
      desc: Design coffee maker with large water tank.
      water_tank: 1000 # ml
   23: &mercury
      name: {{ arch.brand }} Mercury Maker
      desc: Compact maker for small espresso only.
      water_tank: 350 # ml
   29: 
      <<: *mercury
      name: {{ products.22.name }} Deluxe Edition
...
```

The description of the coffee capsules is given with the `capsules.yml` file. The limited edition capsules are located into another file to make easier diff across versions. This second file is merged into the main `capsule.yml` with a `<>` directive. 
```
# capsules.yml
---
<>:
   merge: {filename: 'capsules_limited.yml'}
   
capsules:   
   - name: Roma
     temperature: 89
     pressure: 17.5
     color: brown
     id: 0x2918CF
     
     
   - name: Escobar
     temperature: 73
     pressure: 15
     color: purple
     id: 0xACF23D     
```
