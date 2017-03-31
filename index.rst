#####################################
Pessimistic pattern matching for LSST
#####################################

Abstract
========

The current implimentation of the object matcher for astrometry has been found to not be exceptionly robust and fails to find matchs on serveral current datasets. This document will summarize those failures and suggest improvements that can be made to the current matching algorithm.

Introduction
============

The Large Synopitic Survey Telescope (LSST) will amass the largest astronomical dataset to date when it beings taking data in the next decade. High precision absolute astrometry and photometric are required to exploit the full potential of these data. Futher complicating this, the LSST images are required to reduced, calibrated, and differenced against previous observations to search for transient objects. All this must take place within 60 seconds of the image being taken.

Underpinning our ablity reach the astrometric and photomtric calboration goals are how quickly and accurately we can match the sources we detect in a raw image to an internal or external (e.g. GAIA) reference catalog given likely errors in telescope pointing or ditortion models. The LSST software currently uses an implimentation of [tabur07]_`s Optimsitic Pattern Matcher B (OPMb) designed for use with Hyper Suprime-Cam (HSC) data. This algorithm attemps to find match N point pinwheels between detection and reference catalogs, exiting on the first pattern it finds. There are several failures of this current implimentation. First, many internal varriables that set matching tolerances for this algorithm are set by hand with no documentation as to why they are set so. Secondly and more importantly, the dynamic range of the current implimentation is not sufficiant to handle both observations high Galactic latitude where the stellar density is low and observations in the Galactic plain where the stellar density is orders of magnitudes higher. As LSST will observe at both high and low Galactic latitude, it is important that any matching algorithm perform relyably without any user intervention in both conditions.

In this document we present modifications to the OPMb algorithm that we dub Pessimistic Pattern Matcher B (PPMb). PPMb differs from the previous algorithm in that we have both the option to require a consensus of several matched patterns before it considers a rotation and shift in the estimated World Coordinate System (WCS) is taken. This is crutial for high stellar density regions where finding patterns between the detection and source catalog is not rare. PPMb also simplified the number of tolerances significantly and automaticly calculates these tolerances from the data where possible. The result is a pattern matcher that that works for over a much larger dynamic range of stellar density with very little sacrific of computation time.

This document is divided as follows. First we introduce the OPMb algorithm and our modifications to create PPMb. Second, we describe the datasets we use to verify the performance of our matcher and what ever tuning we performed to run across this large dynamic range. We then present the Results after running on a large dataset from 3 different datasets from current instruments and surveys. Lastly we summarize our work and make suggestions for possible future improvements to the algorithm.

Method overview
===============

In this section we describe the modifications and simplifications we make to OPMb to create PPMb.

Pessimsitic Pattern Matcher B
-----------------------------

Like OPMb, PPMb relies on matching N point pinwheels for a detection or source catalog into a catalog of reference objects. The allows the matching to account for both shifts and rotations in the WCS with some allowance for distortion or scaling. Searching for such pinwheel shapes rather than triangles allows efficient creation of and searching for patterns "on the fly" rather than precomputing all the :math:`n (n - 1) (n - 2) / 6` unique triangles available in a reference catalog with :math:`n` elements. Instead the algorithm need only precompute the :math:`n (n - 1) / 2` unique pairs between reference objects. In [[tabur07]_` OPMb is tested only up to 100's of reference objects in a given observation. In the Galactic plain, using the GAIA catalog as a reference, there can be ~5 thousand of reference objects in a roughly 100 sq. arcminute section of the sky with a similar or even greater number of detected sources. At this point in the reduction process we do not have access to a photometric calibration and thus trimming the number of reference objects based on flux is not very motivated This is esspecially true considering a large faction of the brightest sources will be saturated in the image thus have poor centroids.

The first difference between OPMb and PPMb is found in the space these pairs are computed and compared. Both algorithms require an initial WCS guess with OPMb using this WCS to project the reference catalog positions onto the CCD tangent plain. PPMb insetad uses the initial WCS, RA and DEC positions for the sources compares them to the references on the unit sphere. We chose this to avoid complications with the poles in equatorial coordiantes and to avoid problems associated with projections. Working on the unit sphere also means that our final shift and rotation in the WCS can be computed from dot- and cross-products for the spokes and also avoids the need for computationly costly square roots and inverse trig functions.

Algorithm step-by-step
----------------------

The PPMb algorithm beings by creating the data structures needed to both search for individual pattern spokes based on their distance and futher compare the opening angle between different spoke lengths. We each reference pair we precompute the vector deltas: :math:`v_A - v_B`, distances: :math:`|v_A - v_B|`, catalog IDs of the objects that make up the pair. Each of these arrays is sorted by distance. We also store a lookup table of indicies that allows the code access all the pairs featuring a reference with a given ID while preserving the pair distance ordering.

We begin the matching first by sorting our source catalog in magnitude from brightest to faintest. The motivation for this is that the brightest objects are also the rarest and more likely to be uniquely matched between the source and reference catalogs. To attempt the matching we 

Automated tolerances
--------------------

Test datasets
=============

To test the performance of the pessimistic matcher we use it within the context of several current datasets available to us. These data span a range of stellar density and quality of optical distortion models. We process these data in the context of the LSST Stack version 13.

CFHTLS
------

HITS
----

New Horizons
------------

Results
=======

Comparison to Optimistic Pattern Matcher B
------------------------------------------

Summary
=======

