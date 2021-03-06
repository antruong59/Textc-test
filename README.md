# Textc

**Textc** is a natural language processing library that allows developers build text command based applications.

[![Build status](https://ci.appveyor.com/api/projects/status/dd24q1u410h3nrct?svg=true)](https://ci.appveyor.com/project/Take/textc-csharp) <a href="https://www.nuget.org/packages/Takenet.Textc" rel="NuHet">![NuGet](https://img.shields.io/nuget/v/Takenet.Textc.svg)</a>


## How it works


**Textc** does the **tokenization** of text inputs by looking for **matches** in a collection of **syntaxes**, which are grouped in a **text processor**.
The engine tries to parse the input using the defined token types in each syntax.
When there's more than one syntax match, the processing engine choses the best match by using a **scorer**.

With the selected syntax, an **expression** that holds the parsed tokens is generated, being submitted to a **command processor**. 
The most common command processor implementation binds expressions tokens to (class/delegates) methods arguments, thought a conversion of the token type to the language (C#) type.

Optionally, the command processor can produce an output (for instance, the bound method can have a return value), which is handled by an **output processor**.

### Context

Beside the text input, the engine can also use the **request context** to fulfill a syntax token requirement. 
The context is a dictionary of name-value variables and its idea is to act like a *natural* conversation context. 

When a person is in a (natural language) conversation at some moments he/she can omit parts of the sentences because its implicit in the conversation context.

For instance:

> John: What brand is your car?

> Paul: My car is a BMW.

> John: **And what color**?

> Paul: It's yellow.

In the second question, John didn't need to specify that he was talking about the car, because it was implicit in the context, and his real question was *Whats the color of your car?*.
So the car "variable" didn't needed to be in the conversation input.
This is the same idea of the Textc context.

It is possible to add something in the context by marking a syntax token as **contextual** or during the command processing.

**Context using Redis** <a href="https://www.nuget.org/packages/Takenet.Textc.Extensions.Redis" rel="NuHet">![NuGet](https://img.shields.io/nuget/v/Takenet.Textc.svg)</a>

To use Redis you must first install the extension package via NuGet.

```
Install-Package Takenet.Textc.Extensions.Redis
```

This package contains the class responsible for using Redis instead of in memory of the application. It is an implementation of IRequestContext interface.

```csharp
return new Textc.Extensions.Redis.RedisRequestContext({RedisEndpoint}, {RedisDatabase}, {RedisTimeout}, {key});
```

### CSDL

The **Command Syntax Definition Language** is a notation that allows the definition of syntaxes in a convenient way.

A CSDL statement is composed by one or more token declarations, each one specified in the following way:

```
name:type(initializer)
```

Where:

* **name** - The name of the token to be extracted, which must be unique in each statement. This value can be used in the binding with method parameters or to store the value in the context. *Optional*.
* **type** - The type of the token to be extracted. The library define some basic types, like `Word` and `Integer`, but the developer can define its own types. *Mandatory*.
* **initializer** - The initialization date for the token type, which is used in specific ways accordingly to the type.  For instance, in the `Word` type, it is used to define the set of words that can be parsed by the token type. In some token types with complex initialization values, this is presented in the JSON format. *Optional*.

For instance, if we have a calculator that must accept commands like `sum 1 and 2`, where `1` and `2` are variable values, the CSDL for this syntax is:

```
command:Word(sum) num1:Integer conjunction:Word(and) num2:Integer
```

We can simplify our syntax by omitting the name of tokens that we known that will not be used in the command execution (will not be bind to a method parameter or stored in the context), like some language constructs. 
In this case, the syntax will be:

```
:Word(sum) num1:Integer :Word(and) num2:Integer
```

We also can define optional tokens in the syntax, meaning that they can be present or not in the input/context, without changing the semantics of the text. 
To notate that, we must put a question mark (`?`) character after the type definition:

```
:Word(sum) num1:Integer :Word?(and) num2:Integer
```

In this case, the syntax can parse inputs like `sum 1 2`, but still `sum 1 and 2`.

You can specify that a token value should be added to the context if the syntax is matched, by using the plus (`+`) character after the token name declaration (which in this case is mandatory), like this:

```
command+:Word(sum) num1:Integer :Word?(and) num2:Integer
```

Now, if there's a match for this syntax, the `command` variable will be added to the context with the token value (`sum`) and if next user input is something like `1 and 2` or even `1 2`, this syntax will be matched. This kind of token is called **contextual**.


By default, the syntax parsing is done from left to right and a match can happen even if still is some text input to be consumed. 
For instance, our previous syntax will match `sum 1 and 2 and 3 and 4`, but only the first two numbers will be considered.
To avoid that, we can add boundaries to the syntax surrounding it with the `[]` characters like this:

```
[command+:Word(sum) num1:Integer :Word?(and) num2:Integer]
```

We can also change the initial parsing direction of the syntax, by adding an anchor, which is represented by the circumflex (`^`) character in the start or the end of the syntax (in the start is not required, since by default the parsing is left to right).
To parse from the right to left, we do:

```
[command+:Word(sum) num1:Integer :Word?(and) num2:Integer]^
```

And we can change the direction during the parsing, by annoting the token type with a tilde (`~`) character, like this:

```
[command+:Word(sum) num1:Integer :Word?~(and) num2:Integer]^
```

In this case, the parsing starts from the right and if there's a match of the `and` word, the parse of the input will continue from the left, with the first non-matched token (`command` in this example).

This is useful when you have `Text` token types (which are greedy) in the syntax. If you need to match inputs like `translate ol?? mundo to english`, you should have a syntax like this:

```
[:Word?(translate) text:Text :Word~(to) language:Word(english,portuguese)]^
```

And you will have the value `ol?? mundo` assigned to the `text` token and the `english` value to the `language` token.


### Basic token types


| Name     | Description                         | Sample values       |
| ---------|-------------------------------------|---------------------|
| Decimal  | A culture-invariant decimal number. |  `-1.123`,`0.1231`,`1.1`, `131.2` |
| Integer  | A 32 bit number.                    |  `-1`,`0`,`1`, `300` |
| Long     | A 64 bit number.                    |  `-1`,`0`,`1`, `292908889192986509` |
| Word     | A text word. It consumes all the input until the next blank space character. You can limit the expected words in the token template initialization.  Multiple words enclosed within single quotes will be recognized as a single word. | `Dog`, `cat`, `Banana`    |
| Text     | A text with multiple words. Note that this token type will consume all remaining input, so it must be the last to be parsed in a syntax.              | `This is a sentence`      |


### Special token types


| Name      | Description                         | Sample values       |
| ----------|-------------------------------------|---------------------|
| LDWord    | A [levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) word. It parses any word with a default distance of 2 characters from the initialization values. | `Mispelled` for `Misspelled` |
| DMWord    | A [double metaphone](https://en.wikipedia.org/wiki/Metaphone) word. It matches if the generated metaphone code for the input is the same for any of the initialization values. | `Ekzampul` for `Example` |
| RegexText | A text with an regex initializer. Note that this token type will consume all remaining input that matches the specified regex. | `Mycustom text`    |
| RegexLong | A 64 bit number an regex initializer. It allows more flexible parsing rules. | `4582379237123`    |
| RegexWord | A text word with an regex initializer. It allows more flexible parsing rules. | `custom-word-pattern`    |
| SWord     | A [soundex](https://en.wikipedia.org/wiki/Soundex) word. It matches if the generated soundex code for the input is the same for any of the initialization values. | `Ekzampul` for `Example` |


## Samples

### Calculator


Creating the text processor:

```csharp
// Fist, define the calculator methods
Func<int, int, Task<int>> sumFunc = (a, b) => Task.FromResult(a + b);
Func<int, int, Task<int>> subtractFunc = (a, b) => Task.FromResult(a - b);
Func<int, int, Task<int>> multiplyFunc = (a, b) => Task.FromResult(a * b);
Func<int, int, Task<int>> divideFunc = (a, b) => Task.FromResult(a / b);
            
// After that, the syntaxes for all operations, using the CSDL parser:
                        
// 1. Sum:            
// a) The default syntax, for inputs like 'sum 1 and 2' or 'sum 3 4'
var sumSyntax = CsdlParser.Parse("operation+:Word(sum) a:Integer :Word?(and) b:Integer");
// b) The alternative syntax, for inputs like '3 plus 4'
var alternativeSumSyntax = CsdlParser.Parse("a:Integer :Word(plus,more) b:Integer");

// 2. Subtract:            
// a) The default syntax, for inputs like 'subtract 2 from 3'
var subtractSyntax = CsdlParser.Parse("operation+:Word(subtract,sub) b:Integer :Word(from) a:Integer");
// b) The alternative syntax, for inputs like '5 minus 3'
var alternativeSubtractSyntax = CsdlParser.Parse("a:Integer :Word(minus) b:Integer");

// 3. Multiply:            
// a) The default syntax, for inputs like 'multiply 3 and 3' or 'multiply 5 2'
var multiplySyntax = CsdlParser.Parse("operation+:Word(multiply,mul) a:Integer :Word?(and,by) b:Integer");
// b) The alternative syntax, for inputs like '6 times 2'
var alternativeMultiplySyntax = CsdlParser.Parse("a:Integer :Word(times) b:Integer");

// 4. Divide:            
// a) The default syntax, for inputs like 'divide 3 by 3' or 'divide 10 2'
var divideSyntax = CsdlParser.Parse("operation+:Word(divide,div) a:Integer :Word?(by) b:Integer");
// b) The alternative syntax, for inputs like '6 by 2'
var alternativeDivideSyntax = CsdlParser.Parse("a:Integer :Word(by) b:Integer");

// Define a output processor that prints the command results to the console
var outputProcessor = new DelegateOutputProcessor<int>((o, context) => Console.WriteLine($"Result: {o}"));
            
// Now create the command processors, to bind the methods to the syntaxes
var sumCommandProcessor = new DelegateCommandProcessor(
    sumFunc,
    true,
    outputProcessor,                
    sumSyntax, 
    alternativeSumSyntax
    );
var subtractCommandProcessor = new DelegateCommandProcessor(
    subtractFunc,
    true,
    outputProcessor,
    subtractSyntax,
    alternativeSubtractSyntax
    );
var multiplyCommandProcessor = new DelegateCommandProcessor(
    multiplyFunc,
    true,
    outputProcessor,
    multiplySyntax,
    alternativeMultiplySyntax
    );
var divideCommandProcessor = new DelegateCommandProcessor(
    divideFunc,
    true,
    outputProcessor,
    divideSyntax,
    alternativeDivideSyntax
    );

// Finally, create the text processor and register all command processors
var textProcessor = new TextProcessor();
textProcessor.CommandProcessors.Add(sumCommandProcessor);
textProcessor.CommandProcessors.Add(subtractCommandProcessor);
textProcessor.CommandProcessors.Add(multiplyCommandProcessor);
textProcessor.CommandProcessors.Add(divideCommandProcessor);

```

Submitting the input:

```csharp
Console.Write("> ");

try
{
    await textProcessor.ProcessAsync(Console.ReadLine(), context);
}
catch (MatchNotFoundException)
{
    Console.WriteLine("There's no match for the specified input");
}
```

Results:

```
> sum 2 and 3
Result: 5

> 12 plus 3
Result: 15

> subtract 30 from 90
Result: 60

> 77 minus 27
Result: 50

> multiply 2 and 3
Result: 6

> 6 times 6
Result: 36

> divide 30 by 5
Result: 6

> 100 by 10
Result: 10

> sum two and three
There's no match for the specified input

```


### Calendar

Creating the text processor:

```csharp
// 1. Define the calendar syntaxes, using some LDWords for input flexibility
var addReminderSyntax = CsdlParser.Parse(
    "^[:Word?(hey,ok) :LDWord?(calendar,agenda) :Word?(add,new,create) command:LDWord(remind,reminder) :Word?(me) :Word~(to,of) message:Text :Word?(for) when:LDWord?(today,tomorrow,someday)]");
var partialAddReminderSyntax = CsdlParser.Parse(
    "^[:Word?(hey,ok) :LDWord?(calendar,agenda) :Word?(add,new,create) command+:LDWord(remind,reminder) :Word?(for,me) when+:LDWord?(today,tomorrow,someday)]");
var getRemindersSyntax = CsdlParser.Parse(
    "[when:LDWord?(today,tomorrow,someday) :LDWord(reminders)]");
            
// 2. Now the output processors
var addReminderOutputProcessor = new DelegateOutputProcessor<Reminder>((reminder, context) =>
{
    Console.WriteLine($"Reminder '{reminder.Message}' added successfully for '{reminder.When}'");
});
var getRemindersOutputProcessor = new DelegateOutputProcessor<IEnumerable<Reminder>>((reminders, context) =>
{
    var remindersDictionary = reminders
        .GroupBy(r => r.When)
        .ToDictionary(r => r.Key, r => r.Select(reminder => reminder.Message));

    foreach (var when in remindersDictionary.Keys)
    {
        Console.WriteLine($"Reminders for {when}:");

        foreach (var reminderMessage in remindersDictionary[when])
        {
            Console.WriteLine($"* {reminderMessage}");
        }

        Console.WriteLine();
    }
});
    
// 3. Create a instance of the processor object to be shared by all processors
var calendar = new Calendar();

// 4. Create the command processors
var addRemiderCommandProcessor = new ReflectionCommandProcessor(
    calendar,
    nameof(AddReminderAsync),
    true,
    addReminderOutputProcessor,
    addReminderSyntax);

var partialAddRemiderCommandProcessor = new DelegateCommandProcessor(
    new Func<string, Task>((when) =>
    {
        Console.Write($"What do you want to be reminded {when}?");
        return Task.FromResult(0);
    }),
    syntaxes: partialAddReminderSyntax);

var getRemidersCommandProcessor = new ReflectionCommandProcessor(
    calendar,
    nameof(GetRemindersAsync),
    true,
    getRemindersOutputProcessor,
    getRemindersSyntax);


// 5. Register the the processors
var textProcessor = new TextProcessor();
textProcessor.CommandProcessors.Add(addRemiderCommandProcessor);
textProcessor.CommandProcessors.Add(partialAddRemiderCommandProcessor);            
textProcessor.CommandProcessors.Add(getRemidersCommandProcessor);

// 6. Add some preprocessors to normalize the input text
textProcessor.TextPreprocessors.Add(new TextNormalizerPreprocessor());
textProcessor.TextPreprocessors.Add(new ToLowerCasePreprocessor());

return textProcessor;

```

The calendar class:

```csharp
public class Calendar
{
    private readonly List<Reminder> _reminders;

    public Calendar()
    {
        _reminders = new List<Reminder>();
    }

    public Task<Reminder> AddReminderAsync(string message, string when, IRequestContext context)
    {            
        var reminder = new Reminder(message, when);
        _reminders.Add(reminder);
        context.Clear();
        return Task.FromResult(reminder);            
    }

    public Task<IEnumerable<Reminder>> GetRemindersAsync(string when)
    {
        if (string.IsNullOrEmpty(when))
        {
            return Task.FromResult<IEnumerable<Reminder>>(_reminders);
        }

        return Task.FromResult(
            _reminders.Where(r => r.When.Equals(when, StringComparison.OrdinalIgnoreCase)));
    }

    public class Reminder
    {
        public Reminder(string message, string when)
        {
            Message = message;
            When = when ?? "someday";
        }

        public string Message { get; }

        public string When { get; }
    }
}

```

Submitting the input:

```csharp
Console.Write("> ");

try
{
    await textProcessor.ProcessAsync(Console.ReadLine(), context);
}
catch (MatchNotFoundException)
{
    Console.WriteLine("There's no match for the specified input");
}
```

Results:

```
> remind me to write some unit tests for the library
Reminder 'write some unit tests for the library' added successfully for 'someday'

> calendar, remind to pay my bills today
Reminder 'pay my bills' added successfully for 'today'

> renind to learn to write in english tomorou
Reminder 'learn to write in english' added successfully for 'tomorrow'

> remind   me  to   fix  my   keyboard
Reminder 'fix my keyboard' added successfully for 'someday'

> add reminder
What do you want to be reminded ?
> to drink a coffee
Reminder 'drink a coffee' added successfully for 'someday'

> to drink a beer
There's no match for the specified input

> remind me tomorrow
What do you want to be reminded tomorrow?
> to do the laundry
Reminder 'do the laundry' added successfully for 'tomorrow'

> reminders
Reminders for someday:
* write some unit tests for the library
* fix my keyboard
* drink a coffee

Reminders for today:
* pay my bills

Reminders for tomorrow:
* learn to write in english
* do the laundry


> todai reminders
Reminders for today:
* pay my bills

```

## TODO

- [x] Import code from private repository
- [ ] Write new unit tests
- [ ] Better documentation
- [ ] Multiple culture support
- [ ] Language specific token types (Verb, subject, conjunction)
- [ ] Language specific common syntaxes repository
