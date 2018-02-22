#####################################
Pessimistic pattern matching for LSST
#####################################

Abstract
========

The current implementation of the object matcher for astrometry has been found to not be exceptionally robust and fails to find matches on several current datasets. This document will summarize those failures and suggest improvements that can be made to the current matching algorithm.

Introduction
============

The Large Synoptic Survey Telescope (LSST) will amass the largest astronomical dataset to date when it beings taking data in the next decade. High precision absolute astrometry and photometric are required to exploit the full potential of these data. Further complicating this, the LSST images are required to reduced, calibrated, and differenced against previous observations to search for transient objects. All this must take place within 60 seconds of the image being taken.

Underpinning our ability reach the astrometric and photometric calibration goals are how quickly and accurately we can match the sources we detect in a raw image to an internal or external (e.g. GAIA) reference catalog given likely errors in telescope pointing or distortion models. The LSST software currently uses an implementation of [tabur07]_`s Optimistic Pattern Matcher B (OPMb) designed for use with Hyper Suprime-Cam (HSC) data. This algorithm attempts to find match N point pinwheels between detection and reference catalogs, exiting on the first pattern it finds. There are several failures of this current implementation. First, many internal variables that set matching tolerances for this algorithm are set by hand with no documentation as to why they are set so. Secondly and more importantly, the dynamic range of the current implementation is not sufficient to handle both observations high Galactic latitude where the stellar density is low and observations in the Galactic plain where the stellar density is orders of magnitudes higher. As LSST will observe at both high and low Galactic latitude, it is important that any matching algorithm perform reliably without any user intervention in both conditions.

In this document we present modifications to the OPMb algorithm that we dub Pessimistic Pattern Matcher B (PPMb). PPMb differs from the previous algorithm in that we have both the option to require a consensus of several matched patterns before it considers a rotation and shift in the estimated World Coordinate System (WCS) is taken. This is crucial for high stellar density regions where finding patterns between the detection and source catalog is not rare. PPMb also simplified the number of tolerances significantly and automatically calculates these tolerances from the data where possible. The result is a pattern matcher that that works for over a much larger dynamic range of stellar density with very little sacrifice of computation time.

This document is divided as follows. First we introduce the OPMb algorithm and our modifications to create PPMb. Second, we describe the datasets we use to verify the performance of our matcher and what ever tuning we performed to run across this large dynamic range. We then present the Results after running on a large dataset from 3 different datasets from current instruments and surveys. Lastly we summarize our work and make suggestions for possible future improvements to the algorithm.

Method overview
===============

In this section we describe the modifications and simplifications we make to OPMb to create PPMb.

Pessimsitic Pattern Matcher B
-----------------------------

Like OPMb, PPMb relies on matching N point pinwheels for a detection or source catalog into a catalog of reference objects. The allows the matching to account for both shifts and rotations in the WCS with some allowance for distortion or scaling. Searching for such pinwheel shapes rather than triangles allows efficient creation of and searching for patterns "on the fly" rather than pre-computing all the :math:`n (n - 1) (n - 2) / 6` unique triangles available in a reference catalog with :math:`n` elements. Instead the algorithm need only pre-compute the :math:`n (n - 1) / 2` unique pairs between reference objects. In [[tabur07]_` OPMb is tested only up to 100's of reference objects in a given observation. In the Galactic plain, using the GAIA catalog as a reference, there can be ~5 thousand of reference objects in a roughly 100 sq. arcminute section of the sky with a similar or even greater number of detected sources. At this point in the reduction process we do not have access to a photometric calibration and thus trimming the number of reference objects based on flux is not very motivated This is especially true considering a large faction of the brightest sources will be saturated in the image thus have poor centroids.

The first difference between OPMb and PPMb is found in the space these pairs are computed and compared. Both algorithms require an initial WCS guess with OPMb using this WCS to project the reference catalog positions onto the CCD tangent plain. PPMb instead uses the initial WCS, RA and DEC positions for the sources compares them to the references on the unit sphere. We chose this to avoid complications with the poles in equatorial coordinates and to avoid problems associated with projections. Working on the unit sphere also means that our final shift and rotation in the WCS can be computed from dot- and cross-products for the spokes and also avoids the need for computationally costly square roots and inverse trig functions.

Algorithm step-by-step
----------------------

The PPMb algorithm beings by creating the data structures needed to both search for individual pattern spokes based on their distance and further compare the opening angle between different spoke lengths. We each reference pair we pre-compute the vector deltas: :math:`v_A - v_B`, distances: :math:`|v_A - v_B|`, catalog IDs of the objects that make up the pair. Each of these arrays is sorted by distance. We also store a lookup table of indicies that allows the code access all the pairs featuring a reference with a given ID while preserving the pair distance ordering.

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

+-------------------+-------------------+-------------------------------+----------------+
|                      HSC New Horizons (pointing=908), 4056 CCDs                        |
|                                Median Reference: 5442                                  |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.02) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       3863        |             0.952             |       10       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |        464        |             0.114             |       0        |
+-------------------+-------------------+-------------------------------+----------------+

+-------------------+-------------------+-------------------------------+----------------+
|                       DECam HiTS (183 visits), 10980 CCDs                              |
|                                Median Reference: 167                                   |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |      10213        |             0.930             |      640       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       8979        |             0.818             |      1724      |
+-------------------+-------------------+-------------------------------+----------------+

+-------------------+-------------------+-------------------------------+----------------+
|                     CFHTLS g, r-band (325 visits), 11700 CCDs                          |
|                                 Median Reference: 96                                   |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       11182       |             0.956             |      176       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       11335       |             0.967             |      108       |
+-------------------+-------------------+-------------------------------+----------------+

+-------------------+-------------------+-------------------------------+----------------+
|                          CFHTLS u-band (56 visits), 2016 CCDs                          |
|                                 Median Reference: 92                                   |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       1957        |             0.971             |       13       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       1943        |             0.964             |       19       |
+-------------------+-------------------+-------------------------------+----------------+

+-------------------+-------------------+-------------------------------+----------------+
|                          CFHTLS i-band (56 visits), 2016 CCDs                          |
|                                 Median Reference: 96                                   |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       1932        |             0.958             |       12       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       1959        |             0.972             |       8        |
+-------------------+-------------------+-------------------------------+----------------+

+-------------------+-------------------+-------------------------------+----------------+
|                          CFHTLS z-band (56 visits), 2016 CCDs                          |
|                                 Median Reference: 91                                   |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       1973        |             0.979             |       9        |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       1994        |             0.989             |       7        |
+-------------------+-------------------+-------------------------------+----------------+


+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|                                                    All solved CCDs                                                    |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|                           | N Matched CCDs | Mean Scatter [arcsec] | Median Scatter [arcsec] | Sigma Scatter [arcsec] |
+===========================+================+=======================+=========================+========================+
|   NH: MatchPessimisticB   |      4046      |         0.020         |          0.008          |         0.088          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|   NH: MatchOptimisticB    |      4056      |         1.183         |         1.2860          |         0.4452         |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|  HiTS: MatchPessimisticB  |     10340      |         0.016         |          0.014          |         0.035          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|  HiTS: MatchOptimisticB   |      9256      |         0.011         |          0.011          |         0.005          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
| CFHTLS: MatchPessimisticB |     11524      |         0.065         |          0.061          |         0.159          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
| CFHTLS: MatchOptimisticB  |     11592      |         0.064         |          0.062          |         0.036          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+

+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|                                                    5 Sigma clipped                                                    |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|                           | N Matched CCDs | Mean Scatter [arcsec] | Median Scatter [arcsec] | Sigma Scatter [arcsec] |
+===========================+================+=======================+=========================+========================+
|   NH: MatchPessimisticB   |      3850      |         0.008         |          0.008          |         0.001          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|   NH: MatchOptimisticB    |      4052      |         1.184         |          1.286          |         0.444          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|  HiTS: MatchPessimisticB  |     10126      |         0.015         |          0.014          |         0.005          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
|  HiTS: MatchOptimisticB   |      8965      |         0.011         |          0.011          |         0.004          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
| CFHTLS: MatchPessimisticB |     11233      |         0.061         |          0.061          |         0.012          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+
| CFHTLS: MatchOptimisticB  |     11531      |         0.063         |          0.062          |         0.015          |
+---------------------------+----------------+-----------------------+-------------------------+------------------------+

+---------------------------+---------------------+-----------------------+----------------------+
|                                     Method Timing Comparison                                   |
+---------------------------+---------------------+-----------------------+----------------------+
|                           | Mean time [seconds] | Median time [seconds] | Sigma time [seconds] |
+===========================+=====================+=======================+======================+
|   NH: MatchPessimisticB   |       86.126        |        15.996         |      112.800         |
+---------------------------+---------------------+-----------------------+----------------------+
|   NH: MatchOptimisticB    |       68.690        |        12.347         |      123.853         |
+---------------------------+---------------------+-----------------------+----------------------+
| CFHTLS: MatchPessimisticB |        0.616        |         0.566         |        0.239         |
+---------------------------+---------------------+-----------------------+----------------------+
| CFHTLS: MatchOptimisticB  |        0.516        |         0.498         |        0.150         |
+---------------------------+---------------------+-----------------------+----------------------+


Summary
=======
