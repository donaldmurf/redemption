# Should we repair false positives?

Clang-tidy has a checker called `cppcoreguidelines-init-variables` which is very prolific. It reports 9120 of the 9157 EXP33-C alerts for Git, and 5215 of the 5225 EXP33-C alerts for Zeek.

This checker checks if any variables aren't initialized at the point where they are defined, but EXP33-C is only violated if variables still aren't initialized by the time they are first read. As a result, many of this checker's alerts are false positives, in the sense that while they indicate potential read of an uninitialized variable, no such read actually takes place.

This brings into question: should we be repairing false positives in general?  Repairing a false positive without diagnosing it as such is a central tenet to the Redemption project.

## Copyright

<legal>
'Redemption' Automated Code Repair Tool
Copyright 2023, 2024 Carnegie Mellon University.
NO WARRANTY. THIS CARNEGIE MELLON UNIVERSITY AND SOFTWARE ENGINEERING
INSTITUTE MATERIAL IS FURNISHED ON AN 'AS-IS' BASIS. CARNEGIE MELLON
UNIVERSITY MAKES NO WARRANTIES OF ANY KIND, EITHER EXPRESSED OR IMPLIED,
AS TO ANY MATTER INCLUDING, BUT NOT LIMITED TO, WARRANTY OF FITNESS FOR
PURPOSE OR MERCHANTABILITY, EXCLUSIVITY, OR RESULTS OBTAINED FROM USE OF
THE MATERIAL. CARNEGIE MELLON UNIVERSITY DOES NOT MAKE ANY WARRANTY OF ANY
KIND WITH RESPECT TO FREEDOM FROM PATENT, TRADEMARK, OR COPYRIGHT
INFRINGEMENT.
Licensed under a MIT (SEI)-style license, please see License.txt or
contact permission@sei.cmu.edu for full terms.
[DISTRIBUTION STATEMENT A] This material has been approved for public
release and unlimited distribution.  Please see Copyright notice for
non-US Government use and distribution.
This Software includes and/or makes use of Third-Party Software each
subject to its own license.
DM23-2165
</legal>

## Why we should repair false positives
### Generates less SA alerts

In theory, code that is repaired due to some SA alert should cause the SA tool to not re-generate the same alert on the repaired code. So repairing false positives will reduce the number of alerts generated by the SA tool.

This theory could always be false; a SA tool may still issue alerts for repaired code, and one of Redemption's design principles is to not repair code that has already been repaired. Assuming that Redemption did the repair correctly, such a case would be best addressed in the next section.

Generating less SA alerts saves an auditor time and effort, and reduces the cost of maintaining that an alert is a false positive through evolution of the codebase.

### False positives should be pruned by the SA tool, not by Redemption

Static analysis of source code is a 40-year-old industry, with many commercial and open-source SA tools available. As false-positive alerts have been a problem for SA tools, some have striven to minimize the false-positive alerts they report.

By comparison the Redemption project is a 2-year line project, and we do not expect our tool to be better at identifying false positive alerts than the current state-of-the-art SA tools.  Consequently, we are disinclined to add detection of false positives to the Redemption tool. There are three alternative approaches we would recommend:

1. Use a better SA tool, with a lower false positive rate. (FPR)
2. Tweak the SA tool to lower its FPR.  Many tools will provide interfaces or API's for doing this.
3. Do filtering for false positives outside of Redemption and the SA tool.

Nonetheless, some false positives are easy to recognize in Redemption. When it is cheap and easy to do so reliably, Redemption can identify some false positives and refuse to repair them.

## Why we should NOT repair false positives
### Makes code harder to read

The one generalization I would suggest here is that code that you last modified is more comprehensible than code that someone else last modified.  Code that is repaired by the Redemption tool is therefore less comprehensible to a human being than that same code before the repair. This is a fundamental problem that we cannot solve.

Otherwise, the ease of reading code is a very subjective measure, and should be addressed on a case-by-case basis. One thing we could do is to annotate every repair with some comment, such as "/* ACR */", to indicate this code was automatically repaired.

### Makes code slower

Adding additional checks for null pointers or other problems does impede performance. We try to minimize the performance hit, typically by wrapping a check inside a single if statement. EG:

    if (error_condition) {
      // handle error
    }
    // do original behavior

Evaluating the error condition should be cheaper than doing the original behavior. For example, checking if a pointer is null is just as cheap as dereferencing it.  Likewise, the additional if statement imposes a minimal performance hit when the error condition does not occur. The performance hit when an error does occur can be greater, but that indicates an error condition, indicating more serious problems than performance.

In theory, a sufficient optimizing compiler can eliminate the performance hit imposed by a repair if the alert was a false positive. For example, if a compiler can infer that a pointer can not be null when the null check occurs, it could eliminate the null check.

### Dead Code

Repairing an alert that is known to be a false positive typically creates dead code, or code with no effect. For example, a pointer that is reported as possibly being NULL adds a null check, and error-handling code.  If the alert is known to be a false positive, then the null check is always false, and the error-handling code is effectively dead. Dead code is a violation of CERT recommendation [MSC12-C](https://wiki.sei.cmu.edu/confluence/x/5dUxBQ).

There are two partial rebuttals to this argument. The first is the subjectivity of said knowledge. A SA tool that reports an alert clearly does not "know" that it is a false positive.  An alert may be known as a false positive to a cybersecurity expert, but not to a novice. 

The second rebuttal is that while introducing code that violates CERT recommendation MSC12-C is a liability, it is deemed worthwhile when the alternative is to potentially violate a CERT rule, like dereferencing a null pointer. ([EXP34-C](https://wiki.sei.cmu.edu/confluence/x/QdcxBQ)). 

### Can obscure other errors

Consider this code:

``` c
void do_init(int *);  // always initializes the parameter
void g(int, int);
void f(int x) {
    int y;  // uninitialized
    do_init(&y);
    g(x, y);
}
```

An alert that complains that `y` is uninitialized would be repaired by setting `y` to 0, even if the alert is a false positive.

However, imagine a future change where `do_init(&y);` is replaced with this:

``` c
    if (x > 0) {
        do_init_a(&y);
    }
    if (x < 0) {
        do_init_b(&y);
    }
```

It would then have a different, real defect, currently un-handled by Redemption. That is CERT recommendation [MSC01-C. Strive for logical completeness](https://wiki.sei.cmu.edu/confluence/display/c/MSC01-C.+Strive+for+logical+completeness), which warns against logically incomplete code. This code is logically incomplete in that it does not handle the case where `x` is 0.

Consequently, the only way `y` could remain uninitialized is if `x == 0`, a condition not captured by both if statements. In such a case, reading `y` is undefined behavior.  The repair converts this potential undefined behavior into a (perhaps erroneous) usage of `0`.  While this repair is itself harmless, it obscures the logical bug of what really should happen when `x == 0`.  We do have various tools to catch undefined behavior. Without the repair, most compilers and SA tools could warn about the possibly uninitialized use, and a tool could detect the undefined behavior when it is encountered at run time, so that developers could fix it.  However, repairing the original code leaves the new incomplete logic hole unfixed.

It might even be the case that if `x` is 0, then `y` should also be 0, in which case the code is correct...but we cannot infer this from the code itself.

In theory, a SA tool that could detect the logical incompleteness in the un-repaired code should still be able to detect the incompleteness if the uninitialized variable is repaired.

### Can break code

A core principle of the Redemption project is that repairs should not break code. We prefer caution when making repairs and will perform extensive tests to make sure that our repairs do not break the code.  Therefore, this is a very theoretical reason, in that code breakage would only happen outside the scope of this project.  It is however, an issue that can never completely go away.

## Conclusion

There is no one-size-fits-all solution to this problem. Each SA checker that generates lots of false positives will merit individual attention to this problem, and may demand different solutions.

For `cppcoreguidelines-init-variables`, we will employ the following strategy:

 1. First, try to reduce the number of false positives if we can do so cheaply and effectively. (REM-320)
 2. Disable `cppcoreguidelines-init-variables` from mapping to EXP33-C if it reveals too few true positives to be worth the effort of dealing with the false positives.
 
