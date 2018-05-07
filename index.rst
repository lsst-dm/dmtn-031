#####################################
Pessimistic pattern matching for LSST
#####################################

Introduction
============

The Large Synoptic Survey Telescope (LSST) “Stack” currently uses an implementation of [tabur07]_'s Optimistic
Pattern Matcher B (OPMb) for blind astrometric matching to a reference catalog. This has been found to have a
significant failure mode in very dense stellar fields with ~1000 references per CCD. This is caused by the
algorithm being too greedy and finding false astrometric matches due to the matching space no longer being
sparse at these densities. Rather than fine tune parameters for this algorithm to match successfully, we
generalize the [tabur07]_ algorithm to work consistently over the large range of stellar densities expected in
LSST. In this work we present Pessimistic Pattern Matcher B (PPMb), which operates over the full dynamic range
of densities expected in LSST.

Method overview
===============

In this section we describe the modifications and generalizations we make to OPMb to create PPMb. Like OPMb,
PPMb relies on matching :math:`N` point pinwheel patterns between a source catalog and a catalog of
astrometric reference objects. This allows the matching to account for both shifts and rotations in the WCS
with some allowance for distortion or scaling. Searching for such pinwheel shapes rather than triangles allows
for efficient creation of, and searching for patterns "on the fly" [tabur07]_ rather than pre-computing all of
the :math:`n (n - 1) (n - 2) / 6` unique triangles available in a reference catalog with :math:`n` elements.
Instead the algorithms need only pre-compute the :math:`n (n - 1) / 2` unique pairs between reference objects.
In [tabur07]_ OPMb is tested only up to hundreds of reference objects in a given observation. However, in the
galactic plane, using the Gaia catalog as a reference, there can be of order 5000 reference objects in single
CCD with a similar or greater number of detected sources. This causes challenges for the algorithm as
implemented in the Stack and results in false positive matches causing poor astrometric solutions for these
fields.

Primary Algorithmic Differences in PPMb
---------------------------------------

PPMb follows a very similar algorithmic flow to that of OPMb with several key differences that we list below.

- PPMb matches to the reference catalog on the unit sphere rather than a tangent plane. As implemented in the
  Stack, OPMb relies on an intermediate tangent plane calculated from the mean positions of the astrometric
  references in RA/dec and detected sources in x, y on the CCD to compute matches. PPMb avoids this by
  attempting to find astrometric matches directly on the unit sphere of RA, dec.

- Can require that a specified number of successful pattern matches with the same shift and rotation on the sky
  be found before returning an affirmative match. This is useful in fields with a very high number of
  astrometric reference objects where matching an :math:`N` point pattern at a realistic tolerance is fairly
  common. This allows PPMb to exclude false positive matches even for relatively loose tolerances.

- Allows for matching a pattern of :math:`N` spokes from :math:`M` points where :math:`M` can be greater than
  :math:`(N + 1)`. This allows the matcher skip over objects that may not be in the reference sample due to
  the astrometric reference sample, for instance, being observed in a different bandpass filter.

- Automatically estimates matching tolerances from the input data rather than requiring arbitrary tolerances
  to be set. Arbitrarily setting the tolerance can lead to instances of false positives or long run times as
  the algorithm is forced to fail on all test patterns before softening the match tolerance and trying again.

- Treats both the shift and rotation as matching parameters as known upper bounds. That is the pointing and
  rotation of the telescope are known to within some precision and can be set based on the the known pointing
  imercistion of the telescope the data being processed are from. The current Stack implementation of OPMb
  treats these later of these parameters as not intrinsic but a parameter to be soften.

- The algorithm tightens the maximum shift in addition to matching tolerance as the match-fit cycle of
  the Stack improves. Since the current Stack implementation uses an intermediate tangent plane
  calculated from the mean RA/dec and x,y positions of the references and sources, it is thus unable to
  utilize such a tightening. Tightening the maximum shift allows the code to find a match much more rapidly in
  later iterations of the WCS match-fit cycle.

Algorithm step-by-step
----------------------

The PPMb algorithm beings by creating the data structures needed to both search for individual pattern spokes
based on their distance and further compare the opening angle between different spokes. For each reference
pair we pre-compute the 3-vector deltas: :math:`v_{\Delta}=v_A - v_B`, distances: :math:`|v_A - v_B|`, and
catalog IDs of the objects that make up the pair. Each of these arrays is sorted by the distance. We
additionally store a lookup table of indicies that allows the code to access all the pairs featuring a
reference object with a given ID quickly while preserving the pair distance ordering.

Both OPMb and PPMb construct pinwheel shapes that are used to search within the reference objects from the
detected source catalog by ordering the objects in the catalog from brightest to faintest flux. The pinwheel
is created by choosing the first first :math:`N` brightest objects, treating the brightest of these as the
pinwheel center. If this pattern is not found in the astromtetric references then the brightest source is
discarded and a new N point pinwheel is constructed starting with the second brightest object and so on until
a requested number of patterns have been tested. In [tabur07]_ they suggest testing 50 patterns. Currently the
LSST stack is set by default to test 150.

Shift and rotation tests
^^^^^^^^^^^^^^^^^^^^^^^^

After selecting :math:`N` sources ordered from brightest we compute the :math:`v_{\Delta}` of the first two
brightest points, and search for reference pairs with a distance :math:`|v_R|` that have the same length to
within :math:`\pm \delta_{tol}` of the length of the source vector :math:`|v_S|`. We test each of these
reference pairs from the smallest length difference :math:`\Delta = abs(|v_S| - |v_R|)` to the largest,
assuming that the correct pattern has nearly the same length for the source and length deltas.

These candidate pairs are then tested by first selecting one of the two reference points that make up the pair
as the ``center`` of the pinwheel. By taking the dot product of the two unit-vectors representing the source
center and candidate reference center, we can quickly check if the implied pointing shift is within the
maximum allowed. If the shift is too large we then check against the other point in the reference pair. If
both fail then we move to the reference pair with the next smallest :math:`\Delta` and repeat.

Once a candidate reference pair and reference center are found to within the distance and shift tolerance, we
compute the rotation matrix to align the source and reference centers. We apply this rotation to the source
3-vector rotating it into the reference frame. We then compute the dot-product of this 3-vector with the
candidate reference 3-vector delta to compute the implied rotation of this candidate pair. If it is greater
than our set maximum we continue to the next candidate reference pair.

Pattern construction
^^^^^^^^^^^^^^^^^^^^

Assuming the reference candidate for the two brightest objects in the source pinwheel satisfies all of the
previous tests we began to create the remaining spokes of our :math:`N` point pinwheel. We first pair down the
number of reference pairs we need to search by using the ID lookup table we created previously to select only
reference pairs that contain our candidate reference center. This speeds op the next stages of the search
significantly. As with the first two points we test the length of the vector between the brightest source and
3rd brightest source object against all of the reference pairs that contain the current candidate reference
center. We again sort reference pair candidates from the smallest absolute length difference to the largest.

Once we have the candidates for this source spoke we need only test that the opening angle between this spoke
and initial spoke are within tolerance to the angle formed by the candidate reference objects. We make the
assumption here that the separations between any point in the source or reference objects are small enough
that we can assume that they are within the plane of the sky this simplifying the math needed to compare these
3-vector deltas and allowing use to use the tests described below.

We employ two separate but related tests to test that the opening angle between source pattern spoke we are
testing and the spoke created by the two brightest source objects in the pattern are within tolerance of the
candidate reference spoke we are testing against. Given the length of the source spoke being tested, we create
an angle tolerance by computing

.. math:: \delta_{ang} = \frac{\delta}{L + \delta}

where L is the length of the source spoke. This sets the opening angle tolerance assuming :math:`L >> \delta`
and also simplifies the tolerances that need be specified beforehand. We set a limit that this angle be less
than :math:`0.0447` radians. This is set such that :math:`cos(\delta_{ang}) \sim 1` to within 0.1%. This
allows us to use the small angle expansion of :math:`sin` and :math:`cos` in this opening angle test. For
cases where :math:`L >> \delta` is not held, we instead set the opening angle tolerance to the value
:math:`0.0447`.

To test the opening angle against the current tolerance for this spoke, we compute the normalized dot-product
between our source spoke to the first source spoke and do the same with the candidate reference spokes. We
then test the difference of these two cosines:

.. math:: cos(\theta_{src}) - cos(\theta_{ref})

If we assume that at most :math:`\theta_{src} = \theta_{ref} \pm \delta_{ang}` and Taylor expand for small
values of :math:`\delta_{ang}` then we can write our test as

.. math:: - \delta_{ang} sin(\theta_{ref}) < cos(\theta_{src}) - cos(\theta_{ref}) < \delta_{ang} sin(\theta_{ref})

For computational purposes we square this equation as have not yet computed :math:`sin(\theta_{ref})`. The
test for the difference of cosines is then

.. math:: (cos(\theta_{src}) - cos(\theta_{ref}))^2 < \delta_{ang}^2 (1 - cos(\theta_{ref})^2)

This test on the difference in cosines is not sufficient to know that the two opening angles are the same
within tolerance. To completely test that the angles are within tolerance we also need to test the sine of the
angles. where the previous test first computed the dot-products between source and reference vectors to get
the cosines, we compute the normalized cross-product between the two source spokes and likewise the reference
spokes. This produces vectors with lengths :math:`sin(\theta_{src})` and :math:`sin(\theta_{ref})`
respectively. These vectors can be dotted into the center point of the the respective patterns they are
derived from giving the value of the sine. It should be noted here that the value is approximate as the
vectors are likely slightly misaligned to that of center points,  artificially decreasing the amplitude of the
sine. However, on the scale of a a CCD, the vectors we are comparing should be within the plane of the sky and
thus the comparison holds.

If we again Taylor expand for small angle differences the comparison becomes

.. math:: - \delta_{ang} cos(\theta_{ref}) < sin(\theta_{src}) - sin(\theta_{ref}) < \delta_{ang} cos(\theta_{ref})

These tests in tandem assure us the opening angles are the same between the source and reference spokes and
that they rotate in the same direction. The tests are robust for all values the opening angles for both the
reference and source patterns.

Intermediate verify
^^^^^^^^^^^^^^^^^^^

Once we have constructed the complete pinwheel pattern of the requested complexity, we test that the shift and
rotation implied by the first spoke in each of the source and reference pinwheels can align the reference and
source patterns on top of each other such that the distances between the source and reference points that make
up the pinwheels are all within the matching tolerance. If this condition is satisfied we then fit rotation
matrices using the :math:`N` matched points that transform source objects into the reference frame allowing
for some non-unitarian in the matrix. This matrix will be used to transform the source objects into the
reference frame before running final verify.

Pessimism of the algorithm
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Up until this point PPMb has followed roughly the same algorithmic nature of OPMb though using vectors in
3-space on the unit-sphere instead of on the a focal plane. Once we successfully completed passed intermediate
verify we move compute the translations of several test points from the source frame to the reference frame.
These test points were created by computing the mean 3-vector of the source sample and creating 6 test points
by finding the min and max of each of x,y,z coordinates of the source sample and replacing the x,y,z in the
mean 3-vector of the sources. We do this computation before any attempt to match has been made. Upon finding
a candidate reference pattern we rotate the test points from the source into the reference frame. We then
store these rotated test points and continue our search the next pattern starting with another :math:`N`
point source pinwheel pattern and so on. Once we find more patterns that pass intermediate verify, we rotate
the 6 points again and compare their rotated positions to previous shifts and rotations that have been
matched. If a user specified number of previous shifts and rotations move the test points to within the
:math:`\delta` length tolerance then we can proceed to the final verify step.

We find that finding 3 such matches reduces the false positive rate for dense stellar fields significantly
even for large of :math:`\delta`. We also set a threshold for using this pessimistic mode requiring that both
the number of reference objects and source objects exceeds the total number of source patterns to test before
softening tolerances. This assures us that there are enough objects to have the desired number of matching
patterns.

Final Verify
^^^^^^^^^^^^

Finally, after finding a suitable shift and rotation matrix we apply it and its inverse to the source object
and reference object positions respectively. We construct searchable kd-trees of the source and reference
objects in their respective frames for fast nearest-neighbor look up. After matching the rotated source and
rotated reference objects with the kd-tree we construct a "handshake" match. This matching refers to having
both the sources matched into the reference frame and the reference matched into the source frame agree on
the match in order to consider it valid. This cuts down on false positives in dense fields by requiring that
the source/reference pair are truly the closest. After trimming the matched source and references to the
maximum match distance :math:`\delta`, we test that the number of remaining matches is greater than the
minimum requested. Once this criteria is statisfied we return the matched source and reference catalog.

Automated matching tolerances
-----------------------------

We attempt to guess a good starting match tolerance automatically from the reference and source catalogs. To
do this, we attempt to find the most similar :math:`N` point patterns based on their sorted :math:`N - 1`
spoke lengths. We start by ordering the reference and source catalogs in decreasing flux and creating
:math:`N` point patterns for a total of :math:`n - N` patterns where :math:`n` is the number of objects in
the source or reference catalog. We compute the :math:`N - 1` lengths from brightest object in the pattern to
the fainter ones. We then sort these distances and attempt to find minimal the two patterns out of the
:math:`n - N` total that have the most similar spoke lengths. We then average the distance between over the
:math:`N - 1` spokes. We do this both for the reference and source objects and pick the smaller of the two.
This allows us to set the initial tolerance at a threshold that reduces false positives in the pattern
matching as a function of pattern density.

Softening tolerances
--------------------

PPMb has two main tolerances which can be softened as subsequent attempts are made to match the
source data to the reference catalog. These are the maximum match distance :math:`\delta` and the number of
spokes we allow to fail before moving on to the next center point. We soften the match distance by doubling
it each after the number of patterns requested has failed. We also independently add 1 to number of spokes
allowed to fail. These two softenings allow the algorithm enough flexibility to match to most stellar
densities, cameras, and filters.

Test datasets
=============

To test the performance of the pessimistic matcher we utilize several currently available datasets. These data
span a range of stellar density and quality of optical distortion models. We process these data in the context
of the LSST Stack version 14. It should be noted that this analysis was completed before the merging of
DM-10765 which changed the WCS properties of the stack. For each of these data we use the same set of
reference objects derived from the GAIA DR1 [GAIA CITE] dataset. [HOW MUCH DETAIL SHOULD I PUT INTO THESE
DESCRIPTIONS?]

CFHTLS
------

We use data from the Canada-France-Hawaii Telescope Legacy Survey (CFHTLS) [CFHTLS CITE] observed at the
Canada-France-Hawaii Telescope with MegaCam. The dat come from the W3 pointing of the Wide portion of the
CFHTLS survey. We use a total number of 325 visits (start 704382) in the g and r bands, and 56 visits each in
u (850245), i (705243), and z(850255) filters. This give a total of 17,700 CCD exposures to blindly match.

HITS
----

We use data from the High Cadence Transient Survey (HiTS, [HITS CITE]) observed on the Blanco 4m telescope
with the Dark Energy Camera (DECam). We use observations in the g and r bands and a total of 183 visits
starting with visit id 0406285 for a total of 10,980 CCDs exposures.

New Horizons
------------

We use data that was observed on the Subaru telescope using Hyper-Suprime Cam(HSC) as part of efforts . The
data were observed as part of a path finding effort for the New Horizons probe. There are a total of 39
visits contained in data labeled ``pointing 908`` we we use to test an extremely dense case for both
reference and source objects. This pointing starts with visit id 3350 and contains a total number of 4056 CCD
exposures.

Results
=======

In this section we present results from running the PPMb matching algorithm within the match/fit cycle of
AstronomyTask within the ``meas_astrom`` package of the LSST Stack on the data described previously. We
additionally run the default algorithm OptimisticPatternMatcherB (OPMb) on the same data. We divide the
results into 3 major sections. First we show present the fraction of CCD exposures from each dataset that
found a good astrometric solution. Next we present a comparison of the quality of the matches found by
comparing the RMS scatters between the astrometric solutions found with the two matchers. Finally, we compare
the run times of the two matchers compared between the two datasets. PPMb retains the same configuration
settings throughout while we modify the match tolerance :math:`\delta` for the HSC timing test to give a
fairer comparison with PPMb. OPMb's start tolerance is :math:`3` arcseconds which causes the code to exit
with a false positive match almost instantaneously. We instead set the tolerance to :math:`1` arcseconds for
this test and dataset to more fairly compare the run time with similar starting tolerances between the codes.

Fraction of successful matches
------------------------------

In this section we compare the rate at which PPMb and OPMb are able to find acceptable matches on datasets
spanning different densities of objects, data quality, and bandpass filters. For each dataset we set an
upper-limit on what we consider a successful match/fit cycle based on the expected quality of the astrometric
solution after a successful match. These numbers were derived from confirming successful matches by eye and
noting the RMS scatter in arcseconds of the final astrometric solution. ``N Successful CCDs`` is the number
of CCD-exposures where we find a match and meet this criteria while ``N Failed Match`` are the number of CCDs
where a match to the reference catalog was unable to be found. The success rate is ``N Successful CCDs`` over
the total CCD-exposures available.

CFHTLS Matching
^^^^^^^^^^^^^^^

These data are taken at a high galactic latitude with a limited number of reference objects available to
match to. In addition, the total exposure time of these images (~200 seconds) means that roughly an equal
number of sources are available to match given signal to noise and other quality cuts on the source centroid.

For the largest sample of CCDs we attempted to solve, observed primarily in the g and r bands, the
performance of the two matchers is quite similar, differing only by roughly :math:`1%` in the fraction of CCDs
matched.

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

The same results hold for the 3 remaining bandpasses with both matchers performing to within :math:`1%` of
each other PPMb out performs OPMb in the u-band slightly though like the other two bands this difference is
not significant given the absolute difference in the number of successful matches. Overall, we feel that the
new matching algorithm is performing as well as the one previously implemented on this dataset.

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

High Cadence Transient Survey matching
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For the HiTS data, PPMb outperforms OPMb significantly, with the OPMb algorithm as implemented failing to
find matches for a larger fraction of the CCD-exposures and more low quality matches (scatter > 0.10) than
PPMb.

+-------------------+-------------------+-------------------------------+----------------+
|                       DECam HiTS (183 visits), 10980 CCDs                              |
|                                Median N Reference: 167                                 |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.10) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |      10213        |             0.930             |      640       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |       8979        |             0.818             |      1724      |
+-------------------+-------------------+-------------------------------+----------------+

New Horizons matching
^^^^^^^^^^^^^^^^^^^^^

The New Horizons (NH) data presents the largest challenge for both algorithms. The data is observed within
the Galactic plane and contains a high density of reference objects and detected sources. Complicating the
matching further, many of the brightest reference objects are saturated making them ill suited for use in the
matcher.

The density of objects in this field causes OPMb to perform very poorly. The "optimistic" nature of the
algorithm causes it to exit after finding a false positive match which is easy for the algorithm to find
given the density of reference objects. This is evidenced by the low number of failed matches but the very
high scatter of these matches which is greater than :math:`1` arcseconds. PPMb avoids these false positives
by forcing the algorithm to find 3 patterns that agree on their shift and rotation before exiting and
returning matches.

+-------------------+-------------------+-------------------------------+----------------+
|                      HSC New Horizons (pointing=908), 4056 CCDs                        |
|                                Median N Reference: 5442                                |
+-------------------+-------------------+-------------------------------+----------------+
|      Method       | N Successful CCDs | Success Rate (scatter < 0.02) | N Failed Match |
+===================+===================+===============================+================+
| MatchPessimisticB |       3863        |             0.952             |       10       |
+-------------------+-------------------+-------------------------------+----------------+
| MatchOptimisticB  |        464        |             0.114             |       0        |
+-------------------+-------------------+-------------------------------+----------------+


Match quality comparisons
-------------------------

In addition to the looking at the run number of successfully matched CCDs we also look at the quality of
those matches and the astrometric solutions they produce. We present two tables to summarize these numbers.
First we present the results for all CCDs that were successfully matched and solved by the two algorithms.
For the NH sample, we see that the solutions produced by OPMb are not quality solutions as their RMS scatter
on the solution is greater than several times the pixel scale (:math:`\sim 0.16` arcseconds). PPMb fairs
better here however some solutions still have a large RMS scatter and pull both the mean and variance to
higher values.

For HiTS and CFHTLS the two algorithms are more comparable with PPMb having a slightly large sigma around the
average solution.

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

This table shows the summary statistics computed on the same data as above but now 5 sigma clipped around the
mean to compare the results with outliers removed.

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

Timing match/fit cycle timing
-----------------------------

One concern with the generalizations added to OPMb to make PPMb is if the algorithm can still find matches in
wall clock time comparable to that of the current Stack implementation of OPMb. In this section we present
timing results both for a field with low density and with a high density. We count the time spent matching
from the moment the ``doMatches`` is called till an array of matches (even if it is empty) is returned. We run
through all CCDs in the CFHTLS in the g, r sample run previously and all of the CCD-exposures in NH pointing
908. For both methods there are outliers that heavily skew the mean and variance and thus we clip the times
with a :math:`5 \sigma` iterative clipping.

The timing is both the mean and median suggest that PPMb is between 10% and 30% slower than OPMb for these
datasets. However, it should be noted that PPMb is currently implemented in pure Python using NumPy and
fast searchable data structures where possible. The main pattern creation loop of PPMb relies mostly on
internal Python iteration which can be very slow. This is in comparison the Stack implementation of
OPMb which is coded in C++. The extra steps of PPMb then do not seem to catastrophically increase the
compute time to find astrometric matches.

+---------------------------+---------------------+-----------------------+----------------------+
|                            Method Timing Comparison (5 sigma clipped)                          |
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

In this tech-note, we presented a generalization to the OPMb algorithm from [tabur07]_ that allows for
astrometric matching of catalog of detected sources into a catalog of reference objects in tractable time for
a larger dynamic range of object densities. Such a generalization is important for the denser, galactic
pointings of the LSST dataset. We have shown that the PPMb algorithm to perform similarly both in terms of
match success rate and WCS scatter to that of OPMb in data with a low object density and that it provides
exceptional improvement in fields with a high reference object density. The timing of the two algorithms is
surprisingly similar given that the current Stack implementation of OPMb is written in a compiled language
where as PPMb is currently written in pure Python. Given the performance comparison between the two algorithms
and codes one could switch the default behavior of the LSST Stack to PPMb without any notable drawbacks.
