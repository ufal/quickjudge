# QuickJudge
QuickJudge is a tiny tool to simplify the process of manual judging of string segments (e.g. sentences in machine translation output) of one or more competing systems.

## Download
quickjudge (perl source code, license: CC-BY-SA)
Please, let me (bojar-at-ufal.mff.cuni.cz) know, if you make any use of QuickJudge.

## Using QuickJudge
QuickJudge is used in a three-step annotation process:

1. Combine outputs of all the judged systems to an 'annotation file'.
2. Manually add marks/scores/whatever to the annotation file.
3. Collect marks for individual systems, summarize the results.

### Combine Input Files to Annotation File
QuickJudge expects any number of input files (plaintext or gzipped plaintext).

All the files must have the same number of lines.

Here are four sample files: MT input, reference translation and two hypothesized system outputs:

```
==> in <==
Kočka spí.
Psi štěkají.

==> ref <==
The cat is sleeping.
Dogs bark.

==> outa <==
A kitten yawns.
Dogs barking.

==> outb <==
A cat is sleeping.
Dogs are barking.
```

Use the following command to prepare the annotation file ``my_annotation.ano``:

```bash
quickjudge my_annotation --refs=in,ref outa outb
```
The parameter ``--refs`` is optional and may contain a comma-delimited list of files that should be available to the annotator for a reference.

To de-randomize the annotation file later, QuickJudge creates and uses the file ``my_annotation.coresp``. Make sure to preserve the file until you interpret all your annotations.

### Manually Annotate the Annotation File
This is what QuickJudge prepared for you in ``my_annotation.anot``:

```
in	Kočka spí.
ref	The cat is sleeping.
	A cat is sleeping.
	A kitten yawns.

in	Psi štěkají.
ref	Dogs bark.
	Dogs are barking.
	Dogs barking.
```

Edit the text file using any not overly "clever" text editor. Obey these simple rules:

* Do not hardwrap any lines or change tab characters etc.
* Insert any kind of marks in front of each unlabelled line, e.g. ``**`` to denote best translation, `*` to denote an acceptable one or create custom tokens as you go, such as ``missV`` to mark sentences with a verb missing:

```
in	Kočka spí.
ref	The cat is sleeping.
**	A cat is sleeping.
*	A kitten yawns.

in	Psi štěkají.
ref	Dogs bark.
**	Dogs are barking.
* missV	Dogs barking.
```

Note that the order of sentences is kept intact (unless you ran quickjudge with ``--shuffle``) but the order of the systems is randomized with identical outputs collapsed to a single line.

If there are more than one systems annotated and all the systems produced the same output, the sentence is not printed at all, unless QuickJudge was launched with an additional parameter ``--keep-identical``.

### Collecting Annotations
#### Reading Just Marks
Use the following command to collect and de-randomize the annotations:

```bash
quickjudge --print my_annotation
```
For the sample given above, QuickJudge will print the following summary:

```
1	outb	**
1	outa	*
2	outb	**
2	outa	* missV
```
The output is tab-delimited with the following column meanings:

1. Block (i.e. sentence/segment) number.
2. System name (i.e. the name of the original input file).
3. The annotation you provided.

Use your favourite text processing tools to collect your results, e.g. the number of segments scored with two stars, or the indices of segments where any of the system made an error of a particular type.

#### Reading Whole Lines

Use ``--verbatim`` along with the ``--print`` to obtain full lines, not just the first column of the annotation file. Useful for more complex annotation instructions where people e.g. correct the sentences.

## Several Annotators of the Same Set
You can distribute the annotation file (``.anot``) to any number of annotators, or you can prepare individual annotation files for each of them by independent runs of QuickJudge.

As long as you don't provide the ``.coresp`` file to your annotators, there is no way they can learn which system produced which output.

QuickJudge does not provide any means for combining results from several annotators, use a simple shell for-cycle for that.

## Annotating Only a Random Sub-Sample
If your test set is too big for manual annotation, you may want to annotate only a random subsample of it.

QuickJudge simplifies this with the option ``--shuffle``:

```bash
quickjudge --shuffle my_annotation --refs=in,ref outa outb
```

This will shuffle not only all candidates for a segment but also all segments from the whole file. Simply instruct your annotators to do only the top n items of the ``.anot`` file.
