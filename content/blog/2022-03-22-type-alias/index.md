+++
title = "Type-based Alias Analysis"
[extra]
latex = true
[[extra.authors]]
name = "Andrew Butt"
link = "andrewb1999.github.io"
[[extra.authors]]
name = "Andrey Yao"
link = "https://github.com/andreyyao"
+++

## Introduction
The type system of a statically-typed language allows compilers to reject illegal programs during the type-checking stage of compilation. In a sense, the typing information attached to variables, functions, etc. is a refinement on the set of valid program states, which can be approximated without ever running the program. Although programming language types are often used to locate certain errors during compile time, it's not their only use.

"Type-Based Alias Analysis"[^1] by Diwan, McKinley, and Moss examines how the same principal can be applied to alias analysis, a type of conservative analysis that determines whether two given pointer variables might point to the same memory address. Although there had been prior research on alias analysis, Diwan et al.'s type-based alias analysis (TBAA) has the following advantages:
* It is flow-insensitive and runs on linear time, as opposed to many other alias analyses which are expensive to compute.
* It performs almost equally well under an open world assumption as it does in closed world. It's compatible with the principle of modular programming.

They also presented evaluations of TBAA. They performed static and dynamic performance analyses on the effectiveness of TBAA when used for redundant load elimination (RLE). Perhaps most notably, they adopted the strategy of limit analysis by comparing empirical speedups with the maximum possible speedups.

In this blog post, we will first study the specific ideas of TBAA with examples in C#-like syntax, as opposed to Modula-3, the language used in the paper. Then we will review the performance analyses and discuss potential factors behind the empirical results. Finally we will discuss how the paper affected later work in the literature.


## Type Preliminaries
**Readers can skip this section if they are already familiar with type systems or wish to focus on TBAA.**

Given two types $\tau_1$ and $\tau_2$, we say that $\tau_1$ is a subtype of $\tau_2$ if whenever a value of $\tau_2$ is expected, it is legal to supply a value of $\tau_1$ in its place. If we view types as sets and all possible values of a type as its elements, the subtyping relation can be considered roughly the subset relation. Familiar examples from Java include `class Person extends Object` and `class LinkedList<T> implements Iterable<T>`, etc. There's also the numeric tower[^2] from Typed Racket:
<p align="center"> <img src="numeric_tower.png" style="zoom:30%;" /> </p>

The subtyping relation is reflexive and transitive. It is an example of a "preorder". We will denote $\tau_1\leq \tau_2$ if the former is a subtype of the latter. There are various ways to construct new subtyping relations given existing ones. For example, if we know that $\tau_1\leq \tau_2$ and $\sigma_1\leq \sigma_2$, it could be reasonable to conclude that the arrow (function) types have $\tau_2\to\sigma_1\leq \tau_1\to\sigma_2$. In this case we say the function type is *covariant* in its return type and *contravariant* in its argument type.

There are other typing rules for constructs like tuples, records, generics, etc., but we will not list all of them here. However, the select examples above already give us a glimpse into the richness of information encoded by types. In general, in a statically-typed type-safe language, stricter typing rules allows more fine-grained TBAA, which we will see shortly.

## Type-Based Alias Analysis
TBAA operates on the program AST instead of the IR. Thus, it has access to higher level information than some other program analyses. Let's assume the language we're working with has the following kinds of memory references:

1. $\tt a.x$ Class field access
2. $\tt a[n]$ Array indexing
3. $\tt*a$ Pointer indirection

An access path is defined to be any combination of one or more of these memory references. For instance, $\tt(*(a.b).c[3])[2][*d[*e.f]]$ is a pathological example of an access path. Basically, access paths are succinct representations of chains of memory references in the AST. We will also define $\tt typeof(\mathcal{P})$ to be the type of the path $\mathcal{P}$.

### Type Declarations Only

To predict whether two paths $\mathcal{P}_1, \mathcal{P}_2$ might alias, an obvious heuristic is to say this is when the set of subtypes of $\tt typeof(\mathcal{P}_1)$ intersects with the set of subtypes of $\tt typeof(\mathcal{P}_2)$ nontrivially. Of course, if the two sets of subtypes are disjoint, then if any expression involving $\mathcal{P}_1$ type checks, the same expression with $\mathcal{P}_2$ substituted in place cannot type check, and so the two paths cannot possibly alias. We will define a function $\tt TD$, which takes two access paths and returns true iff their types have a common subtype.

### With Field Access
We can extend the above heuristic by taking into account the language fact that $\tt a.f$ and $\tt a.g$ cannot alias each other for some object $\tt a$. Here we also assume that a field access and an array indexing never alias. This is probably true for many OOP languages. We can summarize whether two access paths may alias inductively using the following table, where $\tt FTD(\mathcal{P}_1,\mathcal{P}_2)$ is true iff $\mathcal{P}_1$ and $\mathcal{P}_2$ may alias. 

<style type="text/css">
.tg {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;}
.tg th{border-color:black;border-style:solid;border-width:1px;}
.tg .tg-c3ow{border-color:inherit;text-align:center;vertical-align:top}
</style>
<table class="tg">
<tbody>
  <tr>
    <th class="tg-c3ow">$\mathcal{P}_1$</th>
    <th class="tg-c3ow">$\mathcal{P}_2$</th>
    <th class="tg-c3ow">$\tt FTD(\mathcal{P}_1, \mathcal{P}_2)$</th>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p$</td>
    <td class="tg-c3ow">$\tt p$</td>
    <td class="tg-c3ow">$\tt true$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p.f$</td>
    <td class="tg-c3ow">$\tt q.g$</td>
    <td class="tg-c3ow">$\tt f=g \land TD (p,\,q)$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p.f$</td>
    <td class="tg-c3ow">$\tt *q$</td>
    <td class="tg-c3ow">$\tt AT(p.f) \land TD(p.f,\,*q)$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt *p$</td>
    <td class="tg-c3ow">$\tt q[m]$</td>
    <td class="tg-c3ow">$\tt AT(q[m]) \land TD(*p,\,q[m])$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p.f$</td>
    <td class="tg-c3ow">$\tt q[m]$</td>
    <td class="tg-c3ow">$\tt false$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p[n]$</td>
    <td class="tg-c3ow">$\tt q[m]$</td>
    <td class="tg-c3ow">$\tt FTD(p,\,q)$</td>
  </tr>
  <tr>
    <td class="tg-c3ow">$\tt p$</td>
    <td class="tg-c3ow">$\tt q$</td>
    <td class="tg-c3ow">$\tt TD(p,\,q)$</td>
  </tr>
</tbody>
</table>


Here $\tt AT$ stands for "address taken", and $\tt AT(\mathcal{P})$ is defined to be true iff the program has ever taken the address of $\mathcal{P}$. One hidden assumption about the table is that the cases are supposed to be checked from top to bottom. For example, if two paths fit case 2 then case 7 on the last row will not apply. Thus it should be very straightforward to implement the function $\tt FTD$ on an AST recursively using ML-style pattern matching, for example.

### Extended With Assignments
So far $\tt TD$ and $\tt FTD$ operate on the assumption that access paths with compatible types and appropriate field accesses can always read or write to each other. However, this can be improved by observing that given $\tau_1\leq\tau_2$, if there are no assignments from variables of type $\tau_1$ to references of type $\tau_2$ anywhere in the program, then references to type $\tau_1$ cannot possibly alias references to type $\tau_2$. This gives rise to the following algorithm:

```
[Part 1]
//Start by assuming no types alias each other
initiliazie Equiv := {{t} | t is a pointer type}

for each pointer assignment a := b
	EA := set in Equiv containing typeof(a)
	EB := set in Equiv containing typeof(b)
	//Merges the equivalence classes of the two types
	remove EA, EB from Equiv and insert (EA union EB)
end

[Part 2]
for each type t
	ET := set in Equiv containing t
	TypeRefsTable[t] = ET intersect (subtypesof t)
end
```
The algorithm can be broken down into roughly two stages. In Part 1, we construct the equivalence classes of types based on the aliasing relation, which is an equivalence relation. Note that equivalence classes of a set partition the set, so in each loop iteration of the algorithm, each type $\tau$ belongs to a unique set inside `Equiv`. Before we saw any assignments in the program, we assume no types alias each other. Each time we see a pointer assignment, we merge the two classes.

Part 2 of the algorithm refines the equivalence relation with subtyping information, at the cost of symmetry of the relation. When the algorithm terminates we end up with `TypeRefsTable`, a map from types $\tau$ to the set of types that might be referenced by some access path of type $\tau$.

### Asymptotic Complexity
This analysis is flow-insensitive and takes $O(n)$ time, where $n$ is the number of instructions. However, using the result of TBAA can have runtime quadratic in the number of memory reference expressions. 

## Evaluation

The evaluation of TBAA presented in this paper is presented as three different kinds of analysis, static metrics, dynamic metrics, and limit analysis. Eight realistic Modula-3 benchmarks are included in the evaluation.

### Static Evaluation

Static evaluation is the most straightforward analysis method. In the case of TBAA, the static property being evaluated is the number of aliases determined by each of the three forms of TBAA. For each of the benchmarks and for each version of TBAA, the number of local and global alias pairs are calculated. Local alias pairs are heap memory references within the same procedure that may alias, whereas global alias pairs also include references that may alias between procedures.

<p align="center"> <img src="table_5.png" style="zoom:30%;" /> </p>

From static evaluation we can see that the simplest version of TBAA, TypeDecl ($\tt TD$), performs significantly worse than the other versions of TBAA. TypeDecl conservatively says that many more reference pairs may alias. However, simple static evaluation does not give the full picture of the benefits of this algorithm.

### Dynamic Evaluation

In contrast to static evaluation, dynamic evaluation compares the performance of optimizations that use the analysis pass. To evaluate TBAA, the authors implement a Redundant Load Elimination pass using TBAA. Redundant Load Elimination (RLE) is a common optimization that combines versions of loop invariant code motion and common subexpression elimination to hoist memory loads out of loops. Importantly, to determine if a load instruction can be hoisted out of a loop, RLE must know that the memory location being accessed cannot be aliased.

<p align="center"> <img src="table_6.png"/> </p>

As can be seen from the table above, FieldTypeDecl ($\tt FTD$) and SMFieldTypeRefs can significantly improve the number of redundant loads removed during optimization, compared to TypeDecl. However, the improvement in the number of redundant loads eliminated depends on the specific benchmark and is not nearly as big of an improvement as static metrics might suggest. Therefore, the paper concludes that a more precise alias analysis is not necessarily much better for real optimization. Additionally, static metrics are insufficient by themselves for evaluation alias analyses.

### Limit Analysis

While dynamic evaluation provides a better view of the real-world optimization provided by TBAA, both static and dynamic evaluation lack a real comparison point. In theory, TBAA could be overly conservative in all three forms, missing out on significant optimization opportunities. To limit this shortcoming in the analysis, the paper proposes the use of limit analysis. At run-time, the authors track the real number of redundant loads before and after applying RLE. By comparing redundant loads before and after RLE, we can see that in most cases, RLE based on TBAA significantly reduces the fraction of redundant heap references. In fact, most benchmarks show a reduction of redundant heap accesses to within 5% of the upper bound.

<p align="center"> <img src="figure_9.png"/> </p>

The discussion also shows that the majority of remaining redundant loads were a result of specific implementation details. The authors only know of two cases where redundant loads were not eliminated because TBAA did not disambiguate memory aliases.

### Open vs Closed World Assumptions

One advantage of performing alias analysis in type-safe languages like Modula-3 and Java is stronger type-safety assumptions about unavailable code. Most user-defined types are encapsulated within a module, and therefore references to those types cannot be aliased by code outside of the module. Figure 12 below shows that as a result, RLE is minimally impacted by open vs closed world assumptions.

<p align="center"> <img src="figure_12.png"/> </p>

The limitations of alias analysis in unsafe languages like C++ with an open world assumption can be mitigated by a proper understanding of undefined behavior.

### Evaluation Summary

TBAA has surprisingly high accuracy in real-world optimizations while maintaining a fast time bound. The evaluation also effectively shows that extensions to the simple TypeDecl version of TBAA provide significant improvements in accuracy.


## Stengths and Weaknesses of the paper
This TBAA paper took a very concrete approach with examples from Modula-3 syntax, as well as three different performance analysis. One advantage of this is that the paper is fairly self-contained, and doesn't require highly specialized knowledge to understand. Thus the paper is more accessible to researchers from other areas, as well as people who want to implement TBAA but don't have a super strong techincal background. However, as a consequence, the paper doesn't explicitly provide a general theoretical framework that can be easily adopted by future programming languages and concepts.

Limit analysis is a very neat idea, and it should definitely be incorporated into the empirical evaluation of other research projects. Having eight benchmark programs doesn't give the most convincing result, but the fact the programs are all real, practical ones does help.

Finally, what the paper lacks the most is a concrete comparison between the effectiveness of this TBAA against other forms of alias analysis. Although the authors have touched upon how TBAA is usually more efficient, actual numbers and data points would be much more convincing.


## TBAA in the Modern Computing Landscape

The simplicity, speed, and accuracy of TBAA has allowed it to stick around in many modern compilers. LLVM has several different alias analyses, including an implementation of TBAA. The compiler frontend must include type annotations that can then be used for alias based optimization within LLVM itself. With a proper understanding of undefined behavior in C and C++, TBAA has already been implemented for compilers like [LLVM](https://llvm.org/doxygen/TypeBasedAliasAnalysis_8cpp_source.html) and [GCC](https://gcc.gnu.org/onlinedocs/gccint/Alias-analysis.html). Given the continued use of TBAA in real compilers, it is clear that TBAA works well in practice. TBAA can also be used in conjunction with flow sensitive alias analysis to improve accuracy in some cases.


## Related Readings
Markin and Ermolitsky[^3] discussed their implementation of a simple type-based alias analysis, called strict aliasing, for a compiler from C/C++ to Elbrus, a general-purpose VLIW (very long instruction word) processor. Their experimental results show promising speedups in intraprocedural compilation and ok results for program-wide compilation. 

Ireland, Amaral, Silvera, and Cui develeoped "SafeType"[^4], a tool to identify void pointer castings in C/C++. The TBAA paper mentioned that C/C++ language allows unsafe pointer casting and thus requires extra conservative alias analysis, but it didn't really address the issue. SafeType supposedly identifies "violations on the C standard's restrictions on memory accesses" with a flow-sensitive approach and extends TBAA to work on a wider range of C/C++ programs, some of which would have led to unsound aliasing decisions under the original TBAA. The authors remarked that their purely static analysis doesn't introduce runtime overhead, which is problem of EffectiveSan[^5] by Duck and Yap, which adds type safety through dynamically typing C/C++.

Beringer, Grabowski, and Hofmann[^6] proposed a unified framework of pointer analysis for statically typed type-safe languages using "region types". Regions are basically representations of disjoint sets of memories. As opposed to the other implementation-heavy papers, their paper is very type-theoretic, focusing on properties like soundness, decidability, verification, etc.

## References
[^1]: Amer Diwan, Kathryn S. McKinley, and J. Eliot B. Moss. 1998. Type-based alias analysis. SIGPLAN Not. 33, 5 (May 1998), 106–117. [[link]](https://doi.org/10.1145/277652.277670)

[^2]: Vincent St-Amour, Sam Tobin-Hochstadt, Matthew Flatt, and Matthias Felleisen. 2012. Typing the numeric tower. In <i>Proceedings of the 14th international conference on Practical Aspects of Declarative Languages</i> (<i>PADL'12</i>). Springer-Verlag, Berlin, Heidelberg, 289–303. [[link]](https://doi.org/10.1007/978-3-642-27694-1_21)

[^3]: Markin, A., Ermolitsky, A. (2018). Simple Type-Based Alias Analysis for a VLIW Processor. In: Itsykson, V., Scedrov, A., Zakharov, V. (eds) Tools and Methods of Program Analysis. TMPA 2017. Communications in Computer and Information Science, vol 779. Springer, Cham. [[link]](https://doi.org/10.1007/978-3-319-71734-0_9)

[^4]: Ireland, I., Amaral, J. N., Silvera, R., and Cui, S. (2016) SafeType: detecting type violations for type-basedalias analysis of C. Softw. Pract. Exper., 46: 1571– 1588. [[link]](10.1002/spe.2388).

[^5]: Gregory J. Duck and Roland H. C. Yap. 2018. EffectiveSan: type and memory error detection using dynamically typed C/C++. SIGPLAN Not. 53, 4 (April 2018), 181–195. [[link]](https://doi.org/10.1145/3296979.3192388)

[^6]: Lennart Beringer, Robert Grabowski, and Martin Hofmann. 2010. Verifying pointer and string analyses with region type systems. In Proceedings of the 16th international conference on Logic for programming, artificial intelligence, and reasoning (LPAR'10). Springer-Verlag, Berlin, Heidelberg, 82–102. [[link]](https://doi.org/10.1016/j.cl.2013.01.001)
