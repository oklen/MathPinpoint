# MathPinpoint paper

LaTeX source of the paper that introduces MathPinpoint, in the ACL format (`acl.sty` and `acl_natbib.bst` from
[acl-org/acl-style-files](https://github.com/acl-org/acl-style-files)).

Build with [Tectonic](https://tectonic-typesetting.github.io/), which fetches the packages it needs and runs BibTeX itself:

```bash
tectonic main.tex        # writes main.pdf
```

`main.tex` uses the `preprint` option (non-anonymous, with page numbers). Use `review` for an anonymous submission and `final` for a camera-ready version.

| Path | Contents |
|---|---|
| `main.tex` | preamble, title, authors, section order |
| `sections/` | one file per section, plus the appendix |
| `figures/example.tex` | Figure 1, judged candidates of two training queries |
| `figures/pipeline.tex` | Figure 2, how the dataset is built (TikZ) |
| `refs.bib` | bibliography; every entry was checked against DBLP, the ACL Anthology, arXiv or the publisher |
