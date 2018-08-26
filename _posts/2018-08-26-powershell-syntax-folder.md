---
title: Making the PowerShell Syntax Folder
excerpt: The making of the VS Code PowerShell Syntax Folder
category:
  - blog
header:
  overlay_image: /images/header-paper-boat.png
  overlay_filter: 0.50
  teaser: /images/teaser-paperboat.png
tags:
  - vscode
  - powershell
  - extension
---

I had been working on the Puppet Editor Services project for a while and I needed something different, but not too different.  I use a lot of PowerShell, so it was time to give back to the community and help make the PowerShell extension better.  I went through the issue list and one of them caught my eye;

> Collapsible/expandable Functions, Regions, Comment blocks, and Comment based help blocks

[https://github.com/PowerShell/vscode-powershell/issues/1336](https://github.com/PowerShell/vscode-powershell/issues/1336)

It seems that the code folding isn't great, and a new VS Code feature, [syntax code folding](https://code.visualstudio.com/updates/v1_23#_folding-provider-api), would help with that. Also it was a popular request from the community so it was definitely a wanted feature!

This blog post is about my journey to creating the PowerShell VS Code Extension Syntax Folder.  It won't contain deep dives in all of the code, but more how I arrived at the solution and some of the code used to do that.

## Initial thoughts on the solution

According to the [API documentation](https://code.visualstudio.com/docs/extensionAPI/vscode-api#FoldingRange) the folding provider just returns an array of zero or more [FoldingRange objects](https://code.visualstudio.com/docs/extensionAPI/vscode-api#FoldingRange)

**FoldingRange**

- start: number

- end: number

- kind?: [FoldingRangeKind](https://code.visualstudio.com/docs/extensionAPI/vscode-api#FoldingRangeKind)

  - Comment

  - Imports

  - Region

Post blog - The `kind` is optional so you don't have to set it
{: .notice--info}

So each object has a start and end line number, and then a number representing what type of range it is.  VS Code uses this for commands like `Fold all comment regions`.

Now that I knew what information I needed to extract, how would I get that? After doing work on the [Puppet Syntax files](https://github.com/lingua-pupuli/puppet-editor-syntax) I knew that I could use the textmate grammar files to parse a PowerShell script into its grammar tokens.  These tokens could then be used to figure out where the code could be folded.  And because [@omniomi](https://twitter.com/omniomi) had done a lot of work on the [PowerShell syntax files](https://github.com/PowerShell/EditorSyntax) I had a lot of confidence that they could be parsed correctly.


## The first solution

Reference - [Github Pull Request](https://github.com/PowerShell/vscode-powershell/pull/1355)

### Making Hello World

I firstly created a static folding provider, whereby it didn't query anything, but just returned a static list of folding regions. I don't have the code available for this but it looked a little like this:

`src/main.ts`

``` ts
import { FoldingFeature } from "./features/Folding";

...

    extensionFeatures = [

        ...

        new FoldingFeature(documentSelector),
     ];
```

`src/features/Folding.ts`

``` ts
export class FoldingProvider implements vscode.FoldingRangeProvider {
    public async provideFoldingRanges(
        document: vscode.TextDocument,
        context: vscode.FoldingContext,
        token: vscode.CancellationToken,
    ): Promise<vscode.FoldingRange[]> {
        return new vscode.FoldingRange(4, 6, 3);
    }
}

...

export class FoldingFeature implements IFeature {
    private foldingProvider: FoldingProvider;

    constructor(documentSelector: DocumentSelector) {
        this.foldingProvider = new FoldingProvider();
        vscode.languages.registerFoldingRangeProvider(documentSelector, this.foldingProvider);
    }

    public dispose(): any { return undefined; }

    public setLanguageClient(languageclient: LanguageClient): void { return undefined; }
}
```

1. The `Folding.ts` file has a folding Provider (`FoldingProvider`) and Feature (`FoldingFeature`) class.

2. The Provider class generates the Folding Ranges.  In this case I'm using a static list (`return new vscode.FoldingRange(4, 6, 3);`) which generates a single range from line 5 to 7 (Line numbers start at zero in the API) as a comment (3 = Comment range).

3. The Feature class registers the provider within VS Code.

4. In the `main.ts` file, we create the folding feature when the extension starts up.

This is the standard template used in the VS Code PowerShell extension;

`Extension --> Feature --> Provider`

### Loading the grammar and tokens

Source - [Loading the grammar](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/e50704ec811c360b3168328088e71e025e497ffc#diff-bf1f20d4d1ba4c38436fc3def0781bcaR276)

The next thing to do was load the textmate grammar parsing library. In VS Code this comes from the [vscode-textmate](https://github.com/Microsoft/vscode-textmate) npm module.  However loading it was a little difficult.  Fortunately someone had already come across this, and had posted a solution in [VS Code Issue 46281](https://github.com/Microsoft/vscode/issues/46281).  All I did was adapt this code into the Feature class, and we now had a function called [getCoreNodeModule](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/e50704ec811c360b3168328088e71e025e497ffc#diff-bf1f20d4d1ba4c38436fc3def0781bcaR300) which would load `vscode-textmate` ... However, this could only load the module at runtime.  This meant I didn't have access to any of the typescript typings, even though they existed, which was annoying.

Source - [Finding grammar file](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/e50704ec811c360b3168328088e71e025e497ffc#diff-bf1f20d4d1ba4c38436fc3def0781bcaR323)

Now I needed the [PowerShell Textmate grammar file](https://github.com/PowerShell/EditorSyntax).  Unfortunately this file isn't actually distributed in this extension, it comes [vendored directly into VS Code itself](https://github.com/Microsoft/vscode/tree/master/extensions/powershell).  VS Code does have the ability to query loaded extensions so we could go through all of the extensions, looking for the one that contributes a `powershell` grammar file.

``` typescript
     private powerShellGrammarPath(): string {
        // Go through all the extension packages and search for PowerShell grammars,
        // returning the path to the first we find
        for (const ext of vscode.extensions.all) {
            if (!(ext.packageJSON && ext.packageJSON.contributes && ext.packageJSON.contributes.grammars)) {
                continue;
            }
            for (const grammar of ext.packageJSON.contributes.grammars) {
                if (grammar.language !== "powershell") { continue; }
                return path.join(ext.extensionPath, grammar.path);
            }
        }
        return undefined;
    }
}
```

Source - [Creating tokens](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/e50704ec811c360b3168328088e71e025e497ffc#diff-bf1f20d4d1ba4c38436fc3def0781bcaR191)

Lastly, now that we had the grammar file and the grammar parser, we could parse a text document into a series of grammar tokens using the [tokenizeLine](https://github.com/Microsoft/vscode-textmate/blob/master/src/main.ts#L183) function.

### Grammar Tokens

So what do the tokens look like? Given a simple file

``` powershell
  function New-VSCodeShouldFold {
<#
.SYNOPSIS
  Displays a list of WMI Classes based upon a search criteria
.EXAMPLE
 Get-WmiClasses -class disk -ns rootcimv2"
#>
```

When you tokenize the document you get the following tokens.  A token is a startIndex, endIndex and array of scopes.  Note that the text columnn doesn't actually exist on the token, but I added it so you can see what the token is referring to.

| text (*)               | startIndex | endIndex | scopes |
| ---------------------- | ---------- | -------- | ------ |
| `function`             | 0          | 8        | `source.powershell`, `meta.function.powershell`, `storage.type.powershell` |
| ` `                    | 8          | 9        | `source.powershell`, `meta.function.powershell` |
| `New-VSCodeShouldFold` | 9          | 29       | `source.powershell`, `meta.function.powershell`, `entity.name.function.powershell` |
| ` `                    | 29         | 30       | `source.powershell` |
| `{`                    | 30         | 31       | `source.powershell`, `meta.scriptblock.powershell`, `punctuation.section.braces.begin.powershell` |
| `\n`                   | 31         | 32       | `source.powershell`, `meta.scriptblock.powershell` |
| `<#`                   | 32         | 34       | `source.powershell`, `meta.scriptblock.powershell`, `comment.block.powershell`, `punctuation.definition.comment.block.begin.powershell` |

...

## Converting tokens to folding regions

### Braces and parentheses

Source - [Commit](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/09c8adebe135b8f2b39b0035972fd6c96225a315)

If you look at the example above you can see that the the brace character (`{`) has a particular scope name; `punctuation.section.braces.begin.powershell`. In fact this was also true for the closing brace and for parentheses.

| Character | Scope                                         |
| --------- | --------------------------------------------- |
| `{`       | `punctuation.section.braces.begin.powershell` |
| `}`       | `punctuation.section.braces.end.powershell`   |
| `(`       | `punctuation.section.group.begin.powershell`  |
| `)`       | `punctuation.section.group.end.powershell`    |

So to find the foldable regions we need to go through all of tokens looking for the a beginning token, and then continue looking for an ending token.  This would give a simple token pair.  But that wouldn't be enough as the folding regions work with line numbers, not document index.  Fortunately the [VS Code document object](https://code.visualstudio.com/docs/extensionAPI/vscode-api#TextDocument) has a handy helper for this `positionAt`, where you pass in an offset or index and it returns a [`Position` object](https://code.visualstudio.com/docs/extensionAPI/vscode-api#Position) which has a `line` property.

Now we had all the information we needed however there was one problem, what about nested regions, for example;

``` text
$scriptblock = {          <---- There should be folding here
    $hash = @{            <---- And folding here
        'key' = 'value'
    }
}
```

In this case I used a stack to keep track of the state as it processed the tokens.  Whenever it encountered a starting token I added the token to the stack, and when it found an ending token I popped a token off of the stack.

``` text
$scriptblock = {          (1) <---- PUSH 1
    $hash = @{            (2) <---- PUSH 2
        'key' = 'value'
    }                     (3) <---- POP 2
}                         (4) <---- POP 1
```

So (2) and (3) will be paired, and (1) and (4) will be paired.

Because the detection code was extracted into a generic method called [`matchScopeElements`](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/09c8adebe135b8f2b39b0035972fd6c96225a315#diff-bf1f20d4d1ba4c38436fc3def0781bcaR233), I could very easily add detection for both braces and parentheses. And if I ever needed it for other tokens, it would be trivial to add them too.

``` typescript
        // Find matching braces   { -> }
        this.matchScopeElements(
            tokens,
            "punctuation.section.braces.begin.powershell",
            "punctuation.section.braces.end.powershell",
            vscode.FoldingRangeKind.Region, document)
            .forEach((match) => { matchedTokens.push(match); });

        // Find matching parentheses   ( -> )
        this.matchScopeElements(
            tokens,
            "punctuation.section.group.begin.powershell",
            "punctuation.section.group.end.powershell",
            vscode.FoldingRangeKind.Region, document)
            .forEach((match) => { matchedTokens.push(match); });
```

### Here Strings

Source - [Commit](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/f5ee6d2ef1668ee4a22b2e54220122467063f81a)

PowerShell Here strings are multi-line string literals that can either be bounded by `@' .... '@` or `@" .... "@`.  They are a little tricker than the braces because there are no are no start or stop regions.  Instead the starting, ending and middle tokens will contain the `string.quoted.single.heredoc.powershell` (or `string.quoted.double.heredoc.powershell` for the double quoted here string).  So we are looking for contiguous (non-breaking) groups of tokens, for example;

For a PowerShell script;

``` powershell
...
$I = @"
double quoted herestring
"@
Write-Host $I
...
```

It would have the following tokens

| Text                         | Scopes                                         |
| ---------------------------- | ---------------------------------------------- |
|  `= `                        | ...                                            |
| `@"\n`                       | ..., `string.quoted.double.heredoc.powershell` |
| `double quoted herestring\n` | ..., `string.quoted.double.heredoc.powershell` |
| `"@\n`                       | ..., `string.quoted.double.heredoc.powershell` |
| `Write-Host`                 | ...                                            |

In the example above, as we process the tokens in order, we store the starting token when we first see `string.quoted.double.heredoc.powershell` scope.  Then, we check the subsequent tokens to make sure they have the required scope.  When we find a token that doesn't have the required scope, we know this is the end of the block. We can then convert the start and end token to line numbers.

I created a generic function called [`matchContiguousScopeElements`](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/f5ee6d2ef1668ee4a22b2e54220122467063f81a#diff-bf1f20d4d1ba4c38436fc3def0781bcaR264), which takes a list of tokens and a scope name, and returns a list of lines where the contiguous block starts and ends.

``` typescript
        // Find contiguous here strings   @' -> '@
        this.matchContiguousScopeElements(
            tokens,
            "string.quoted.single.heredoc.powershell",
            vscode.FoldingRangeKind.Region, document)
            .forEach((match) => { matchedTokens.push(match); });
         // Find contiguous here strings   @" -> "@
        this.matchContiguousScopeElements(
            tokens,
            "string.quoted.double.heredoc.powershell",
            vscode.FoldingRangeKind.Region, document)
            .forEach((match) => { matchedTokens.push(match); });
```

### Comments

Source - [Commit](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/1e1518068adf0b4b060994fce83349d181d0766c)

There are three types of comments, and each type of comment required a different technique to detect;

Block Comments

``` powershell
<#
Block Comment
#>
```

Region Blocks

``` powershell
#region Region blocks
$something = 'value'
#endregion
```

Contiguous Line Comments

``` powershell
# Line Comment Block
# Line Comment Block
# Line Comment Block
```

#### Block Comments

The Block Comments are the easiest to detect as they have a start and stop token scope; `punctuation.definition.comment.block.begin.powershell` and `punctuation.definition.comment.block.end.powershell`.  In this case we can use the `matchScopeElements` function that we created for the braces and parentheses detection.

``` typescript
        // Find matching block comments   <# -> #>
        this.matchScopeElements(
            tokens,
            "punctuation.definition.comment.block.begin.powershell",
            "punctuation.definition.comment.block.end.powershell",
            vscode.FoldingRangeKind.Comment, document)
            .forEach((match) => { matchedTokens.push(match); });
```

#### Region Blocks

Detecting the region blocks was a little more difficult because they had no unique scope name.  They are just line comments as far as the grammar parser is concerned.  So to do this I instead chose to parse all of the tokens and extract all of the comment lines that start with `region` or `endregion`.

I created a helper function called [`extractRegionScopeElements`](https://github.com/PowerShell/vscode-powershell/blob/1e1518068adf0b4b060994fce83349d181d0766c/src/features/Folding.ts#L395-L419) which does the following;

- Find all of the tokens which are a line comment

- For these tokens, only select line comments which start at the beginning of a line e.g. `$foo = 'bar' # region` will not match

- Now for these tokens, if the line comment text starts with `region` then return a new token with a scope of `custom.start.region`. If the line comment text starts with `endregion` then return a new token with a scope of `custom.end.region`.

Once I had these new tokens, I could then, again, use the `matchScopeElements` function to match the beginning and end of regions.

``` typescript
        // Find matching comment regions   #region -> #endregion
        this.matchScopeElements(
            this.extractRegionScopeElements(tokens, document),
            "custom.start.region",
            "custom.end.region",
            vscode.FoldingRangeKind.Region, document)
            .forEach((match) => { matchedTokens.push(match); });
```

#### Contiguous Line Comments

Contiguous line comments were the most difficult.  As well as detecting line comments, it also needed to ensure the line comments were not broken up

``` powershell
# Line Comment Block   |-- This is the first block
# Line Comment Block   |
$x = 'This will break the comment block'
# Line Comment Block   |-- This is the second block
# Line Comment Block   |
```

To do this, I created the `matchContiguousScopeElements` [helper function](https://github.com/PowerShell/vscode-powershell/commit/ef63eea3e7cab9e08dd36c3664b609912b13f257#diff-bf1f20d4d1ba4c38436fc3def0781bcaR269);

- For each token, find a line comment

- If the next token is also a line comment, then continue processing.  If not, then this is block comment and return the start and end tokens as a match

### Adding a setting to disable the syntax folder

Source - [Commit](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/6a99426f0ff649e8bf73508067074ae6c4f48e6e)

While the syntax folder was probably going to work really well, it did need an option to turn it off.  This meant adding a new configuration option called `powershell.codeFolding.enable` using the following in `package.json`;

``` typescript
        "powershell.codeFolding.enable": {
          "type": "boolean",
          "default": true,
          "description": "Enables syntax based code folding. When disabled, the default ..."
        },
```

I then needed to add some new interfaces to the `settings.ts` [file](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/6a99426f0ff649e8bf73508067074ae6c4f48e6e#diff-f953b3a2fb6d7cd16d698364c6a1dc45) for the new setting name.

And then finally in folding feature [file](https://github.com/PowerShell/vscode-powershell/pull/1355/commits/6a99426f0ff649e8bf73508067074ae6c4f48e6e#diff-bf1f20d4d1ba4c38436fc3def0781bca), if the setting is enabled, then the provider is created and registered, otherwise it doesn't register a provider.

## And then the tests started failing ...

Not long after the initial PR was merged, the integration tests I created started failing, specifically right after VS Code 1.25.0 was released.  Fortunately [Keith Hill](@https://github.com/rkeithhill) found it fairly quickly, and as luck would have it, the vscode-textmate node module had a major version jump from 3 to 4.  Which of course had breaking changes, which broke the Folding Provider.

Keith Hill raised an initial [Pull Request](https://github.com/PowerShell/vscode-powershell/pull/1410) which I then took and added some extra fixtures.  And in no time it was fixed ...

To then find another problem which turned out we had a bad [Typescript Promise](https://github.com/PowerShell/vscode-powershell/pull/1416), which Keith and I fixed quickly too. **Community collaboration For The Win**

## First release !!

Soon after this the folding provider was released in version 1.8.0!

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/Powershell?src=hash&amp;ref_src=twsrc%5Etfw">#Powershell</a> add-in for <a href="https://twitter.com/code?ref_src=twsrc%5Etfw">@code</a> got updated, and folding got a much better grasp of PS syntax. My biggest gripe fixed. Whoever that did that, I salute you :-)</p>&mdash; James O&#39;Neill (@jamesoneill) <a href="https://twitter.com/jamesoneill/status/1017493521367027712?ref_src=twsrc%5Etfw">July 12, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## And then the bug reports started ...

Not too long after, the bug reports starting coming in ...

### Fix code folding on CRLF documents

Issue - [Github Issue #1417](https://github.com/PowerShell/vscode-powershell/issues/1417)

Source - [Pull Request](https://github.com/PowerShell/vscode-powershell/pull/1418)

This was an interesting problem.  The initial issue came in as only Here Strings were not folding correctly, and after some trial and error, found that changing the PowerShell script from CRLF to LF line endings fixed the issue.

So it turns out, how a Regular Expression engine processor interprets the end of line anchor (`$`) changes depending on who the engine is.  That is to say, some engines see CRLF as a line ending whereas some, like NodeJS in VS Code, does not!

> For anchors there's an additional consideration when CR and LF occur as a pair and the regex flavor treats both these characters as line breaks. Delphi, Java, and the JGsoft flavor treat CRLF as an indivisible pair. ^ matches after CRLF and $ matches before CRLF, but neither match in the middle of a CRLF pair. JavaScript and XPath treat CRLF pairs as two line breaks. ^ matches in the middle of and after CRLF, while $ matches before and in the middle of CRLF.

[Reference](https://www.regular-expressions.info/anchors.html)

Also it appeared I was using the vscode-textmate tokeniser incorrectly, and I should've guessed this by the name.  To convert text into grammar tokens I was calling `tokenizeLine`; not `tokenizeDocument` or `tokenizeString`, a _line_.  Going through the VS Code codebase, I found [other instances](https://github.com/Microsoft/vscode/blob/d2b6bbb46fbdf535e2c96b3e00ac56ac1d427a69/src/vs/workbench/parts/themes/test/electron-browser/themes.test.contribution.ts#L131-L162) where things were being tokenised per line, not per document.

So I [changed the tokeniser code to tokenise per line](https://github.com/PowerShell/vscode-powershell/commit/ed119852256c69e104d8da4754ee1f68aa46a4d7), and still return the same tokens as if it was the entire document. And I also added [tests for LF and CRLF files](https://github.com/PowerShell/vscode-powershell/commit/9bca79cfdc5ea48320318327c73ce0b794540085) to make sure the Folding Provider returned the same regions no matter the line ending.

During this I also noticed I didn't actually test for the double quoted here strings, only the single quoted ones, so I added a [quick test](https://github.com/PowerShell/vscode-powershell/commit/f574a43ed058233760cd727667ef534aadee9650) for these as well.

### Make region folding case insensitive and strict whitespace

Issue - [Github Issue #1428](https://github.com/PowerShell/vscode-powershell/issues/1428)

Source - [Pull Request](https://github.com/PowerShell/vscode-powershell/pull/1430)

Another oversight was I was testing with lower case for `region` and `endregion` with region blocks.  But you can specify `Region` and `EndRegion` as well, similar to what was defined in the original [folding regular expressions](https://github.com/Microsoft/vscode/commit/64186b0a262f0ff89a060cf8dbbf8de7ff831a00) in VS Code.  I fixed this in the [Folding Provider](https://github.com/PowerShell/vscode-powershell/blob/c61ca6d7561cb13e23398b8cd17a457f9ab58d8c/src/features/Folding.ts#L425-L426) by adding the case insensitive matcher (`.../i`) to the regular expression.

I also noticed that the regular expression I used to detect regions allowed white space between the hash and the text, for example `#   region` would be a valid starting region.  However the original folding regular expression did not, in fact it required no whitespace at all, that is, only `#region` would be detected as a foldable region.

And yet another oversight was that I was detecting regions which started at the leftmost edge. I could only detect regions which were indented by at least one space.  I fixed this by changing the [empty line detection](https://github.com/PowerShell/vscode-powershell/blob/c61ca6d7561cb13e23398b8cd17a457f9ab58d8c/src/features/Folding.ts#L424) to use `^\s*$` instead of `^\s+$`.

And yet again, I [added tests](https://github.com/PowerShell/vscode-powershell/commit/c61ca6d7561cb13e23398b8cd17a457f9ab58d8c#diff-350f355c3cd6a899b507081fafa7b163) for all of these errors.

### Fix detecting contiguous comment blocks and regions

Issue - [Github Issue #1437](https://github.com/PowerShell/vscode-powershell/issues/1437)

Source - [Pull Request](https://github.com/PowerShell/vscode-powershell/pull/1430)

During fixing the other issues I stumbled upon a different issue (Always the way!).  If I had a script which had the following text, I expected the folding regions to be as so;

``` powershell
# Comment Block 1  --+-- Folding Line 1-3
# Comment Block 1    |
# Comment Block 1  --+
#region                                   --+-- Folding Line 4-9
# Comment Block 2  --+-- Folding Line 5-7   |
# Comment Block 2    |                      |
# Comment Block 2  --+                      |
$something = $true                          |
#endregion                                --+
```

However when I ran the Folding Provider it actually had the following regions;

``` powershell
# Comment Block 1  --+-- Folding Line 1-7
# Comment Block 1    |
# Comment Block 1    |
#region              |  --+-- Folding Line 4-9
# Comment Block 2    |    |
# Comment Block 2    |    |
# Comment Block 2  --+    |
$something = $true        |
#endregion              --+
```

Because the the region comment blocks also appeared as line comments, they were being interpreted incorrectly.  If there were blank lines or other content before the `#region` then the folding was correct.

To fix this issue, I first [refactored the region detection](https://github.com/PowerShell/vscode-powershell/commit/15b6dd1edcc158ffd998a6945323325dc226c4d9) because it was too complex.  I simplified the detection and changed the region detection regular expressions to be more like the original [VS Code definitions](https://github.com/Microsoft/vscode/commit/64186b0a262f0ff89a060cf8dbbf8de7ff831a00).  This made the code easier to maintain in the future; If the region folding changed in VS Code, then the new regular expressions could just be copied directly into the extension.  The refactor resulted in one less line of code, but far more readable.

Now I could make the changes to [fix the original issue](https://github.com/PowerShell/vscode-powershell/pull/1438/commits/09ae74345d99dba5a466b2bdb68264070fa797b2).  I created a new regular expression which could detect a line comment, but only if it wasn't a region begin or end directive; `/\s*#(?!region\b|endregion\b)/i;`

And yet again, I [added tests](https://github.com/PowerShell/vscode-powershell/pull/1438/commits/09ae74345d99dba5a466b2bdb68264070fa797b2#diff-350f355c3cd6a899b507081fafa7b163) for this scenario.

## Second Release !!

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">🎉 <a href="https://twitter.com/hashtag/PowerShell?src=hash&amp;ref_src=twsrc%5Etfw">#PowerShell</a> for VS <a href="https://twitter.com/code?ref_src=twsrc%5Etfw">@code</a> 1.8.2 released! 🎉<br><br>🔸Region folding fix<br>🔸Trailing Whitespace PSSA rule fix<br>🔸Better &quot;Find references&quot; support for variables<br>🔸 Misc 🐛 ☠️<br><br>Thanks to <a href="https://twitter.com/GlennSarti?ref_src=twsrc%5Etfw">@GlennSarti</a>, <a href="https://twitter.com/CBergmeister?ref_src=twsrc%5Etfw">@CBergmeister</a> &amp; <a href="https://twitter.com/r_keith_hill?ref_src=twsrc%5Etfw">@r_keith_hill</a>!!<a href="https://t.co/JLwKvShJpz">https://t.co/JLwKvShJpz</a></p>&mdash; Tyler Leonhardt (@TylerLeonhardt) <a href="https://twitter.com/TylerLeonhardt/status/1022885773094207489?ref_src=twsrc%5Etfw">July 27, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

So far, only a few minor bugs have come but the folder is mostly running as it should!

## Wrapping up

This was a fun little experiment but it did take a lot longer that what I originally thought!!

The full list of Pull Requests is at [Github](https://github.com/PowerShell/vscode-powershell/pulls?q=is%3Apr+folding+is%3Aclosed+author%3Aglennsarti).

### Lessons learnt

1. TESTS ARE IMPORTANT and will save your ass.

2. If you don't know Typescript, things take twice as long.  That's ok because learning takes time, but something to keep in mind.

3. TESTS ARE IMPORTANT and will save your ass.

4. Putting up my code early gave the maintainers ([Tyler](https://twitter.com/TylerLeonhardt), [Rob](https://github.com/rjmholt) and [Keith](@https://github.com/rkeithhill)) plenty of time to comment and shape the direction of this complex feature.  Even though there were a lot of comments (159 at last count), it made it easier for them to finally press the "merge" button because they understood it much better.  Instead of me just throwing up a Pull Request and going "Ta Da".  Communication is important

5. Take the time to document your functions and code.  It helps the project maintainers AND your future self when come back to it a few weeks later

6. Repeat after me; **TESTS ARE IMPORTANT** and will save your ass.
