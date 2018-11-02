:skip-help: true
:css: css/stylesheet.css
:title: How to diff XML

.. footer::

    .. image:: images/shoobx.png

----

How to diff XML
===============

.. class:: name

    Lennart Regebro

.. class:: location

    Plone Conf 2018, Tokyo

----

.. class:: blurb

    Shoobx is the only comprehensive platform for incorporation, employee
    onboarding, equity management, fundraising, board & stockholder
    communication, and more.

.. note::

    I work for Shoobx.

    We do a lot of cool things that involve legal documents,
    and long story short, if you have a Delaware C-Corp, you need us.
    If you are gonna do a startup, you need us.
    Maybe you don't know it yet, but you do.

----

.. image:: images/tenor.gif
    :width: 100%

.. note::

    Although you can print your legal docs and sign them with a pen and then scan them into our system,
    one of our benefits is that you don't have to do that.

----

SBT + magic = PDF
=================

.. note::

    You can create the legal document in our system and sign everything electronically.
    There are workflows for doing all this and filling in documents,
    you can customize them and loads of things I only understand halfway because I'm not a lawyer.

    So we have loads of documents, many of them generated through our own template language,
    unsurprisingly called "SBT" for "Shoobox templates".
    (Psst, it's really mostly RML + ZPT and a bit of magic).

----

.. image:: images/docdocdoc.gif
    :width: 100%

.. note::

    Both documents and templates can have revisions,
    and of course it would be nice to have a way of showing the differences.

----

.. image:: images/docdiff.png
    :width: 100%

.. note::

    Obviously a text diff won't do,
    we need the sort of graphical difference where inserts are shown in green,
    and delets are shown in red and with a strike through.

    The first effort worked, but has less than optimal results.
    It was implemented by someone else than me, and took a month or so.

----

.. note::

    So, this was trickier than we though, so why not use somebodies library?
    So, we took over maintenance of the xmldiff library.
    It existed, seemed to work but was unmaintained, which is why it wasn't
    used from the start.

    I was tasked with implementing document diffing based on xmldiff.

----

.. image:: images/xmldiff-first-effort.png
    :width: 100%

.. note::

    That wasn't hard, but it didn't give nice diffs.

    What you can see here is that instead of inserting a new paragraph three,
    and then changing the numbering, it modifies paragraph three, reinserts it as paraphraph 4,
    and then deletes paragraph 4 and then reinserts it as paragraph five.

    It's a bit unclear, because we also got these unicode error squares,
    which I'll explain later, that's a bug in the code using xmldiff.

----

The output was no good

.. class:: substep

    It was hard to debug

    There was a memory leak in the C code

    I suck at C

    And some infinite loop somewhere

    And it was really hard to improve the matching


.. note::

    But that wasn't the only problem with xmldiff.

    * But the code was nearly unmaintainable, the internal data structure was
      a hierarchical list of lists with parents, ie, each child list had it's
      parent as a part of the list, so if you tried to just print one entry,
      you would print everything anyway. It was confusing.

    * And I suck at C, and this had it's central parts in C, and the Python
      code was very find of one letter variable names, like typical C, so yeah,
      it was hard to read.

    * And there was a memory leak in the C code

    * And some infinite loop somewhere

    * And it was really hard to improve the matching

----

Change Detection in Hierarchically Structured Information
=========================================================

.. note::

    The matching was hard to improve, because the xmldiff library had done
    something clever. Always a dangerous thing.

    The basic algorithm from xmldiff is outlined in a paper with the sexy name
    "Change Detection in Hierarchically Structured Information".

    This paper assumes a very simplified view of a hierachy, where a node has
    a value and a list of children, and that's it. XML doesn't look
    like that. Every node has a tag, can have attributes, and children but also
    contained texts. And the only unique ID you can count on is the XPath.

    So, the xmldiff library did something clever, it transformed the tree of
    complex nodes to a tree of simple nodes.

----

.. code:: xml

    <para section="3">
        <b>3. </b>Lorem ipsum have some gypsum
    </para>

.. image:: images/event_tree1.png
    :width: 60%

.. note::

    Now every node has an type, a value, and a list of children.
    This is much more similar to how the paper on hierarchical diffing works,
    so it then gets easy to apply the algorithm.

----

.. code:: xml

    <para section="4">
        <b>4. </b>Lorem ipsum have some gypsum
    </para>

.. image:: images/event_tree2a.png
    :width: 60%

.. note::

    But here we have the same node, except, we have increased the numbering,
    in other words, there was another section inserted before this node.

    But when the library is to make a diff and compare these two nodes,
    it will first notice that the two nodes with numbers have changed,
    those values are different, so the nodes don't match any more.

    Their parent nodes the attribute and the b node, they now have no
    children in common with the old version of the tree.
    So they don't match.

    Which means the top node has two out of three children that do not match,
    so it doesn't match.

----

.. code:: xml

    <para section="4">
        <b>4. </b>Lorem ipsum have some gypsum
    </para>

.. image:: images/event_tree2b.png
    :width: 60%

.. note::

    So to us, these nodes are obviously the same, just different numbering,
    but to xmldiff it was obviously NOT the same node, and the result is
    what we saw before

----

:data-x: r-8000

.. note::

    Whole sections being deleted and reinserted instead of updated.

    So, after trying to fix this for weeks, I started looking into
    writing something from scratch. Something simpler. Diffing can't be
    that hard, can it?

----

:data-x: r9200

Diffing XML is hard!
====================

.. class:: substep

    You can't use normal diffing algorithms

    We want to support two different use cases

    Complex values means difficult comparisons

    Small changes make big differences

    It's slow

----

:data-x: r1200

Longest Common Subsequence
==========================

.. class:: substep

    Old: 1 2 3 4 5 6 7 8

    New: 1 2 9 4 6 5 7 7

    LCS: 1 2 4 6 7

.. note::

    You can't use normal diffing algorithms

    The standard way of diffing linear data like code and text files,
    is something called Longest Common Subsequence.

    Basically you look for bits that are the same and come in the same order.
    This doesn't work, because the data is hierarchical.
    You can use it as a part solution, I'll come to that later, but it doesn't
    work as the main solution.

----

Two different use cases
=======================

.. class:: substep larger

Change management

.. class:: substep

Small diffs

.. class:: substep

Move support

.. class:: substep larger

GUI diff

.. class:: substep

Pretty output

.. class:: substep

Semantically meaningful

.. note::

    There's two cases where you want to diff XML, and the algorithm is
    similar enough in both cases so you don't want to have two different
    implementations. But the usecases are still quite different and have
    different requirements.

    One case is when we want to have a change management system, like some
    sort of source control, but for hierarchical data.

    In that case we want the diffs to be small. We don't care if they make
    semantic sense, just the smallest diff possible, thank you!

    And we also want support for moving data from one tree to another, we
    don't want to have to delete it in one place, and add it in another.

----

Semantically meaningful
=======================

.. class:: substep

    .. image:: images/semanticbad.png

    .. image:: images/semanticgood.png


.. note::

    And the second usecase we want to support is the one I mentioned
    before where we want to see pretty and colorful diffs of the actual
    documents. And there we have two different requirements.
    It should be easy to read, and semantically meaningful.
    Ie, if you change the word "bat" for "car" it should show that in the
    diff, it shouldn't show that the b has been replaced with c and the t with
    an r.


----

:data-y: r800
:data-x: r1200

.. class:: partly3 reveal


----

:data-y: r0
:data-x: r-1200


.. note::

    As we saw in this case, the complexity of the node would make this minor
    change mean a node that should be matched won't be matched. We need to
    look at the node as a whole, but how should different changes be weighed?
    There's no obvious answer, and the algorithm we ended up with is very
    much created by trial and error. Making changes and see what happened.

----

:data-x: r2400

.. class:: partly4

    Small changes in the match can mean big changes in the diff


----

:data-y: r2400



----

* So how to do it

    * General procedure: Compare everything with everything
      Stable marriage problem?

    * But that's gets SLOOOOOW

* Speedups

* Result: Better AND faster, even without C optimizations
