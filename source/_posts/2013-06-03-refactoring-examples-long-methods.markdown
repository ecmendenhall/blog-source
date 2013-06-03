---
layout: post
title: "Refactoring examples: Long methods"
date: 2013-06-03 01:28
comments: true
categories: [Java, refactoring] 
---

Martin Fowler identifies long methods as another common code smell, fixable by breaking one long method into several smaller ones and composing them together. Since _Clean Code_ emphasized writing short, meaningful methods, I had to look a bit to find one. But I'm not very happy with the board coordinate constructor:

{% codeblock lang:java %}
public UniversalBoardCoordinate(String locationPhrase) throws InvalidCoordinateException {
        String noParens = locationPhrase.replace('(', ' ').replace(')', ' ');
        String[] coordinates = noParens.split(",");

        if (coordinates.length != 2) {
            throw new InvalidCoordinateException("That's not a valid board location.");
        }

        row = Integer.parseInt(coordinates[0].trim());
        column = Integer.parseInt(coordinates[1].trim());
    }
{% endcodeblock %}

This is an easy refactor: think about what each line of code does, group the related ones in their own methods, and replace them. In fact, I'd already separated each group with a line break. The first two trim the input string and split it in two:

{% codeblock lang:java %}
private Integer[] parseString(String locationPhrase) throws InvalidCoordinateException {
        String noParens = locationPhrase.replace('(', ' ').replace(')', ' ');
        String[] coordinates = noParens.split(",");
        checkValidity(coordinates);
        return parseCoordinates(coordinates);
}
{% endcodeblock %}

The next two check whether the result is valid:

{% codeblock lang:java %}
private void checkValidity(String[] coordinates) throws InvalidCoordinateException {
        if (coordinates.length != 2) {
            throw new InvalidCoordinateException("That's not a valid board location.");
        }
}
{% endcodeblock %}

{% codeblock lang:java %}
And the last two convert the strings to integers:
private Integer[] parseCoordinates(String[] coordinates) {
        return new Integer[] { Integer.parseInt(coordinates[0].trim()),
                               Integer.parseInt(coordinates[1].trim()) };
}
{% endcodeblock %}

Now that the details of string manipulation are hidden in helper methods, the complicated constructor looks trivial:

{% codeblock lang:java %}
public UniversalBoardCoordinate(String locationPhrase) throws InvalidCoordinateException {
        Integer[] orderedPair = parseString(locationPhrase);

        row = orderedPair[0];
        column = orderedPair[1];
}
{% endcodeblock %}

