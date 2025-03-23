---
title: Migrating from Word/Libreoffice
weight: 5
tags:
  - citation keys
  - word
  - libreoffice
---

So you have decided that enough is enough, and you want to migrate
your existing Word/Libreoffice document to LaTeX/Markdown. Plenty
(well...) tools exist to help with the migration of your document
content, pandoc being the most prominent one, but one thing none
of them will do is keep your citations intact. This will not do.

If you install [this](better-bibtex-citekeys.csl) CSL style in
Zotero (or modify it further and use that), Zotero will render the
in-text citations as `[@citekey]` when you ask Zotero to render the bibliography. If you want LaTeX, modify the
style accordingly. You can then put your document through pandoc or whatnot to get LaTeX/Markdown.

For the curious, BBT does this by patching in the `citation-key`
variable in the CSL processing so it can be rendered using a CSL
style. If you previously used the `citeprocNoteCitekey` preference,
that is now gone, so you'll have to update the style you used.

Update: pandoc now supports docx+citations as input format and will export your word documents into pandoc-compatible markdown with citations! That should be a much smoother experience:

```
pandoc -f docx+citations -t markdown -i Aristotle.docx -o Aristotle.md
```

should do the trick!

--- 

First of all, thank you for your amazing work with BBT. Makes my life much easier!

I ran into problems converting a docx to latex and thought this workaround could help others as well. Pandoc's docx+citations uses the citationId instead of the BBT citekeys (or maybe I missed an option in BBT?).

Here is the native pandoc AST:
```
    , Cite
        [ Citation
            { citationId = "24578"
            , citationPrefix = []
            , citationSuffix = []
            , citationMode = NormalCitation
            , citationNoteNum = 0
            , citationHash = 0
            }
        , Citation
            { citationId = "24580"
            , citationPrefix = []
            , citationSuffix = []
            , citationMode = NormalCitation
            , citationNoteNum = 0
            , citationHash = 0
            }
        ]
        [ Str "[@watkins_posterior_2015;"
        , Space
        , Str "@weatherley_modification_2010]"
        ]
```

So with the help of Claude I created `docx+citations2latex.lua` to get the desired output using
`pandoc -f docx+citations -t latex --lua-filter=docx+citations2latex.lua  -i document.docx -o document.tex`
Maybe this helps also others! 

Content of `docx+citations2latex.lua`:

```
-- Extract citation keys from Cite elements and format as LaTeX citations
function Cite(cite)
    -- Extract citation keys from the text content
    local keys = {}
    local text = ""
    
    -- Concatenate all the text inside the Cite element
    for _, inline in ipairs(cite.content) do
        if inline.t == "Str" then
            text = text .. inline.text
        elseif inline.t == "Space" then
            text = text .. " "
        end
    end
    
    -- Use pattern matching to extract citation keys
    -- Pattern matches anything between @ and ] or , or ;
    for key in text:gmatch("@([%w_%-%.]+)") do
        table.insert(keys, key)
    end
    
    -- If no keys were found, look at the citation IDs as fallback
    if #keys == 0 then
        for _, citation in ipairs(cite.citations) do
            table.insert(keys, citation.citationId)
        end
    end
    
    -- Join keys with commas for LaTeX \cite command
    local keysString = table.concat(keys, ",")
    
    -- Return a LaTeX \cite command
    return pandoc.RawInline("latex", "\\cite{" .. keysString .. "}")
end
```

