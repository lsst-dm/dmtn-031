#####################################
Pessimistic pattern matching for LSST
#####################################

Abstract
========

The current implimentation of the object matcher for astrometry has been found to not be exceptionly robust and fails to find matchs on serveral current datasets. This document will summarize those failures and suggest improvements that can be made to the current matching algorithm.

Introduction
============

The Large Synopitic Survey Telescope (LSST) will amass the largest astronomical dataset to date when it beings taking data in the next decade. High precision absolute astrometry and photometric are required to exploit the full potential of these data. Futher complicating this, the LSST images are required to reduced, calibrated, and differenced against previous observations to search for transient objects. All this must take place within 60 seconds of the image being taken.

Underpinning our ablity reach the astrometric and photomtric calboration goals are how quickly and accurately we can match the sources we detect in a raw image to an internal or external (e.g. GAIA) reference catalog given likely errors in telescope pointing or ditortion models. The LSST software currently uses an implimentation of [tabur07]`s Optimsitic Pattern Matcher B (OPMb) designed for use with HyperSuprimeCam (HSC) data. This algorithm attemps to find match N point pinwheels between detection and reference catalogs, exiting on the first pattern it finds. There are several failures of this current implimentation. First, many internal varriables that set matching tolerances for this algorithm are set by hand with no documentation as to why they are set so. Secondly and more importantly, the dynamic range of the current implimentation is not sufficiant to handle both observations high Galactic latitude where the stellar density is low and observations in the Galactic plain where the stellar density is orders of magnitudes higher. As LSST will observe at both high and low Galactic latitude, it is important that any matching algorithm perform relyably without any user intervention in both conditions.

In this document we present modifications to the OPMb algorithm that we dub Pessimistic Pattern Matcher B (PPMb). PPMb differs from the previous algorithm in that we have both the option to require a consensus of several matched patterns before it considers a rotation and shift in the estimated World Coordinate System (WCS) is taken. This is crutial for high stellar density regions where finding patterns between the detection and source catalog is not rare. PPMb also simplified the number of tolerances significantly and automaticly calculates these tolerances from the data where possible. The result is a pattern matcher that that works for over a much larger dynamic range of stellar density with very little sacrific of computation time.

This document is divided as follows. First we introduce the OPMb algorithm and our modifications to create PPMb. Second, we describe the datasets we use to verify the performance of our matcher and what ever tuning we performed to run across this large dynamic range. We then present the Results after running on a large dataset from 3 different datasets from current instruments and surveys. Lastly we summarize our work and make suggestions for possible future improvements to the algorithm.

Method overview
===============

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

