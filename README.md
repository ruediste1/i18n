[![Build Status](https://travis-ci.org/ruediste/i18n.svg?branch=master)](https://travis-ci.org/ruediste/i18n)

# i18n

This project provides utility classes for internationalization especially useful if you are using Domain Driven Design. The main idea is to associate each localized strings with a java element and use references to the java elements to retrieve the strings. More specifically, a label can be bound to each

* Type
* Property
* Enum Member
* Method
* Method Parameters

In addition, message patterns are bound to message interfaces to allow parameterized string generation.

Care has been taken to allow for static extraction of all definded localized keys, which allows to check if the keys present in some resource files match the keys used by the application.

A fallback translation can be specified directly in code via annotations. This allows simple addition of labels during development, serves documentation purposes and helps translators as example.

## Localized Strings
Localized strings are are represented by **LString**s. Their only capability is to resolve themselves against a **Locale**, resulting in a string representation.

## Labeling Java Elements
To be labeled, a class or enum has to be annotated with the **Labeled** or the **Label** annotation. 

To label a property or method, it has to be annotated with **Labeled** or **Label** or it's containing type has to be annotated with **PropertiesLabeled** respectively **MethodsLabeled**.

To label method parameters, the **Labeled** and **Label** methods can be used on each 
parameter, or the **ParametersLabeled** annotation can be used on the method.  

To label enum members, the containing type has to be annotated with or **MembersLabeled**, respectively. It is not possible to select the enum members to be labeled individually, since enums are often used to represent some kind of state and are passed around in the application. It would be easy for an unlabeled enum member to slip through testing. 

The fallback label is generated from the name in the source code. The names are interpreted in upper camel case for class and enum names, lower camel case for property, method and method parameter names and upper underscore case for enum members. They are converted to upper case, separated with spaces. If the generated fallback label does not fit, the label can be explicitly defined using the **Label** annotation. 

Multiple variants of a label can be specified, by repeating the **Label** annotation, specifying the optional **variant** attribute. For properties, the getter, setter or the backing field can be annotated, but only one of the three per property. For frequently used variants, an annotation can be defined using the **LabelVariant** meta-annotation. The annotation has to have exactly the **value** attribute of string type.

For classes, enums, properties and methods, the available variants are specified implicitly by the present **Label** annotations and additionally by the **variants** attribute of the respective **Labeled**, **PropertiesLabeled** or **MethodsLabaled** annotation.

For enum members exactly the variants specified by the **MembersLabeled** annotation are allowed and available.

## Dealing with Inheritance
To deal with inheritance, class hierarchies are first linearized using [c3 linearization](https://github.com/ruediste/c3java).

If a class or enum is not labeled, the label of the next labeled parent class or interface is used.

The first class to define a property or method defines the label. Derived classes can not change the label and may not use the label annotation. They cannot label an inherited unlabeled property or method.

Enum members are not inherited and therefore there is no fallback for unlabeled enum members.

## Obtaining Labels
The **LabelUtil** is the key class for obtaining labels. It takes a **TranslatedStringResolver** as constructor parameter. (see below). The labels are represented as **TranslatedString**s, which can be resolved against a locale.

For each returned translated string, a key is derived from the fully qualified class or enum name, the property, method or enum member name and the variant and looked up in the resource bundle. In addition, the fallback name is determined.

## Label Lookup
The standard resolver is the **ResouceBundleTranslatedStringResolver**, which uses resource bundles to find locale specific label. If no resource can be found using the key of the translated string, the fallback label is used.

## Message Interfaces
Messages can be accessed by creating an interface annotated with **TMessages**. Each interface method has to return an **LString** or a **TranslatedString** if the method does not take parameters, or an **LString** or a **PatternString** if there are parameters.

Methods may not override each other. The interface may not inherit from other interfaces.

An interface implementation can be generated using **TMessageUtil**. The messages can then be generated by calling the respective methods.

For each method, the localized message is looked up using the fully qualified interface name with the method name as key, using the default message as fallback. If there are no parameters, the message is returned as-is. Otherwise it interpreted as pattern and resolved against the parameters.

The default message is generated from the method name. It is interpreted as lower camel. The first character is converted to upper case, the words are separated with spaces and the message is terminated with a period. If this does not fit (which is likely), the default message can be specified explicitly using the **TMessage** annotation.


## Message Format
The **MessageFormat** is a replacement for the class of the same name from the standard library with the following features

* familiar syntax
* new format types can be defined
* existing format types can be replaced
* use a PEG and a parser library to keep parsing code manageable
* support for the plural concept from ICU

See javadoc for more information.

## Additional Resource Keys
It is possible to define resource keys without a corresponding java element (type, property, ...). Simply create a subclass of `AdditionalResourceKeyProvider` and implement it. The maven mojo (see below) will pick it up. During application startup, use the `AdditionalResourceKeyCollector` to collect the additional keys and 
have your `TranslatedStringResolver` take them into account. When using the `ResouceBundleTranslatedStringResolver`, you can simply pass the additional keys to it.

## Workflow
A maven plugin is used to generate a properties file containing the default translation of all labels and message interfaces in all allowed variants. Add the following to the `build/plugins` of your `pom.xml`

     <plugin>
          <groupId>com.github.ruediste.i18n</groupId>
          <artifactId>i18n-maven-plugin</artifactId>
          <version>1.0-SNAPSHOT</version>
          <executions>
               <execution>
                    <phase>process-classes</phase>
                    <goals>
                         <goal>generate-resource-file</goal>
                    </goals>
                    <configuration>
                    <basePackage>test</basePackage>
                    </configuration>
               </execution>
          </executions>
     </plugin> 

Using a resource bundle editor, the different languages can be synchronized (for example using the [Eclipse ResourceBundle Editor](http://essiembre.github.io/eclipse-rbe)). 

When performing a move or rename refactoring, the "Update fully qualified names in non-java text files" option should always be selected. This automatically keeps the properties files in sync with the code.

When sending properties files to translators, only committed versions should be used. The file should be renamed to include the commit hash. When file is returned, the original original commit should be checked out and the file replaced. Then the changes can be merged/rebased into the development branch. This results in a good merge tooling.


