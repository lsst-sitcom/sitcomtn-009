..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This tech note describes the command structure of the AOS CSCs, including M1M3, M2, and the M2 and Camera Hexapods.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

############
Introduction
############

This tech note describes the command structure of the Active Optics System (AOS) CSCs.
We also discuss the variable names that are relevant to AOS operations.
The AOS consist of the follow CSCs

- M1M3 https://github.com/lsst-ts/ts_m1m3support
- M2 https://github.com/lsst-ts/ts_mtm2_cell
- M2 and Camera Hexapods https://github.com/lsst-ts/ts_mthexapod
- MTAOS https://github.com/lsst-ts/ts_MTAOS
- ASC https://github.com/lsst-ts/ts_alignmentSystemClient

The M1M3 here refers to the M1M3 support system only, not including the M1M3 thermal system (https://github.com/lsst-ts/ts_M1M3Thermal).
The latter is also important for the system image quality, but does not directly interact with the AOS, nor the support system.
The same applies to the camera rotator (https://github.com/lsst-ts/ts_mtrotator), and the
Vibration Monitoring System (VMS https://github.com/lsst-ts/ts_vms).
The VMS has components on M1M3, M2, the camera rotator, and the Telescope Mount Assembly (TMA).

The alignment system client (ASC) is used at the beginning of every night to help put the system into the capturing range of the closed-loop AOS.
It is therefore discussed here as well.

The command structure and the naming of the variables are defined by the Interface Control Documents (ICDs): LTS-161 (M1M3), LTS-162 (M2), LTS-160 (hexapods) and MTAOS (LTS-163??).
Their implementations are found in the XML:

- M1M3 https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTM1M3
- M2 https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTM2
- M2 and Camera Hexapods https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/Hexapod
- MTAOS https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTAOS
- ASC https://github.com/lsst-ts/ts_xml/tree/tickets/DM-28520/sal_interfaces/MTAlignment

Or, in more human-readable format:

- M1M3 https://ts-xml.lsst.io/sal_interfaces/MTM1M3.html
- M2 https://ts-xml.lsst.io/sal_interfaces/MTM2.html
- M2 and Camera Hexapods https://ts-xml.lsst.io/sal_interfaces/MTHexapod.html
- MTAOS https://ts-xml.lsst.io/sal_interfaces/MTAOS.html
- ASC https://ts-xml.lsst.io/sal_interfaces/MTAlignment.html

Each CSC also has its own documentation page:

- M1M3 https://ts-m1m3support.lsst.io/v/develop/index.html
- M2 https://github.com/lsst-ts/ts_mtm2/blob/master/README.md
- M2 and Camera Hexapods https://ts-mthexapod.lsst.io
- MTAOS https://ts-mtaos.lsst.io
- ASC (??)

.. MTAOS architecture and design considerations are also discussed in `tstn-026 <https://tstn-026.lsst.io/>`__.

This purpose of this tech note is to describe the logic behind the design, and how the commands and variables are intended to be used.
We only focus on the force related commands and variables for M1M3 and M2, and position related commands and variables for the hexapods.
There are a lot more commands and variables for each CSC that are not covered here.

####
M1M3
####

On the top level, the M1M3 actuator forces, including x, y, and z-forces, fall into four categories.

#. Applied forces, including forces applied by the AOS and other forces applied by the user during testing. The command to apply the forces is ``cmd_applyForces``. The input to the command is an 1D array of size 156. Each element of the array is a z-force values to be applied to a specific actuator. The actuators are ordered with ascending IDs, as found `here <https://github.com/lsst-ts/ts_m1m3support/blob/0a4fff4c4bb9bef82b73c5e04aeb79a268cf45bf/SettingFiles/Tables/ForceActuatorTable.csv>`__. Every time the user-supplied forces are applied, there is an event ``evt_appliedForces`` which publishes the forces being applied. The input forces are aggregate forces, instead of forces on top of the existing forces. Therefore, a new ``cmd_applyForces`` command overrides all previous ``cmd_applyForces`` commands.

#. Force-balance (FB) forces. The FB system is disabled by default, and can be enabled using command ``cmd_enableForceBalance``. Once enabled, the FB forces are automatically applied based on the net forces and moments measured by the hard points. When the FB forces change, events ``evt_balanceForces`` are published. The published parameters include the 12 x-forces, 100 y-forces, 156 z-forces, and the x, y, and z net forces and moments.


#. Slewing forces. The forces needed for slewing the M1M3 together with the TMA are applied automatically by the control system. Therefore there is no separate command for applying these forces. These forces are calculated based on measurements from onboard accelerometers (not the VMS accelerometers) and gyros. They are published as events ``evt_accelerationForces`` and ``evt_velocityForces``.

#. Look-Up Table (LUT) forces. The M1M3 LUT forces have the following components:

  - Elevation dependency. The corresponding force component is ``evt_elevationForces``. Note that the LUT component due to force actuator internal weights is included here, because it is elevation dependent. See `Sec. 5 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#actuator-internal-weight-components/>`__ for more details.
  - Azimuth dependency. The corresponding force component is ``evt_azimuthForces``.
  - Temperature dependency. The corresponding force component is ``evt_thermalForces``.
  - Static bending forces. The corresponding force component is ``evt_staticForces``.
  - Load cell zero offset. The corresponding force component is ``evt_loadCellZeroOffset``. To be implemented. See `Sec. 10 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#load-cell-zero-offsets/>`__.

The command ``cmd_applyForces`` is not only intended for closed-loop AOS. It is also supposed to be used for manual AOS test, and bump test, where an arbituary set of forces need to be applied.

The total applied forces, which is the sum of the above categories, are published as ``evt_totalForces``.
The events ``evt_totalCylinderForces`` give the same information, but all the forces are converted into cylinder forces instead of x, y, and z-forces.

The forces, as measured by the load cells, are given as telemetry under the topic ``tel_forceActuatorData``.
Under this topic, the user can find 156 ``primaryCylinderForces``, 112 ``secondaryCylinderForces``, 12 ``xForces``, 100 ``yForces``, 156 ``zForces``, and the net x, y, z-forces and x, y, z-moments.

The M1M3 LUT forces are on whenever the mirror control system is in the enabled state.
Due to delay in TMA telemetry, by default, M1M3 uses its own inclinometer reading as input to the elevation LUT.
Only when the elevation angle from the onboard inclinometer is not available would the TMA telemetry be used.
The source for the elevation angle can be switched between the onboard inclinometer and the TMA telemetry using the
command ``cmd_selectElevationSource``, which takes one input parameter - 1=Onboard, 2=MTMount.
The change in the elevation source triggers the event ``evt_elevationSource``.

The determination of the slewing forces uses the onboard accelerometers and gyro by default.
It only falls back to the TMA telemetry when due to some reason the readings from the onboard instruments are not available.

##
M2
##

Similarly, for M2, the total forces could have the following components -

#. Active optic forces. The command for applying active optic forces is ``cmd_applyForces``. The command takes two 1D arrays as inputs, one for axial forces, with size 72, and the other for tangent forces, with size 6. The ordering of the axial actuators goes as B1, B2, ..., B30, C1, C2, ..., C24, D1, D2, ..., D18. For the tangent links it is A1, A2, ..., A6. The forces applied by this command are kept track of using events ``evt_appliedForces.axial`` and ``evt_appliedForces.tangent``, containing 72 axial forces and 6 tangent forces, respectively.  The input forces are aggregate forces, instead of additional forces on top of existing forces. Therefore, a new ``cmd_applyForces`` overrides all previous ``cmd_applyForces`` commands.

#. Force-balance (FB) forces. The FB system is disabled by default, and can be enabled using command ``cmd_enableForceBalance``. Once enabled, the FB forces are automatically applied based on the net forces and moments measured by the hard points. When the FB forces change, events ``evt_balanceForces`` are published. The published parameters include 72 axial forces and 6 tangent forces and the x, y, and z net forces and moments.

#. Look-Up Table (LUT) forces. The M2 LUT forces have the following components:

  - Elevation dependency. The corresponding force component are ``evt_elevationForces.axial`` and ``evt_elevationForces.tangent``. Note that the LUT component due to force actuator internal weights is included here, because it is elevation dependent. See `Sec. 5 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#actuator-internal-weight-components/>`__ for more details. In Harris' language, the elevation dependent components include FE and FA. See `this notebook <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a03_LUT_0.ipynb/>`__.
  - Azimuth dependency. The corresponding force component are ``evt_azimuthForces.axial`` and ``evt_azimuthForces.tangent``. (To be implemented)
  - Temperature dependency. The corresponding force component are ``evt_thermalForces.axial`` and ``evt_thermalForces.tangent``.
  - Static bending forces. The corresponding force component is ``evt_staticForces.axial`` and ``evt_staticForces.tangent``. In Harris' lauguage, this includes `F0 <https://github.com/lsst-ts/ts_mtm2_cell/blob/master/configuration/lsst-m2/config/parameter_files/luts/FinalOpticalLUTs/F_0.csv/>`__ and `FF <https://github.com/lsst-ts/ts_mtm2_cell/blob/master/configuration/lsst-m2/config/parameter_files/luts/FinalOpticalLUTs/F_F.csv/>`__.
  - Load cell zero offset. The corresponding force component is ``evt_loadCellZeroOffset``. To be implemented. See `Sec. 10 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#load-cell-zero-offsets/>`__.

The command ``cmd_applyForces`` is not only intended for closed-loop AOS. It is also supposed to be used for manual AOS test, and bump test, where an arbituary set of forces need to be applied.

There are no forces for slewing, because M2 slews passively. The actuators are much stiffer than M1M3. When the TMA slews, the M2 hard points will sense the load and the FB system will offload the forces and moments.

The forces, as measured by the load cells, are given as telemetry under ``tel_forceActuatorData.axial`` and ``tel_forceActuatorData.tangent``.

The M2 LUT forces are on whenever the mirror control system is in the enabled state.
By default, the elevation angle used by the elevation dependent component of the LUT is from the onboard inclinometer.
Only when the elevation angle from the onboard inclinometer is not available would the TMA telemetry be used.
The source for the elevation angle can be switched between the onboard inclinometer and the TMA telemetry using the
command ``cmd_selectElevationSource``, which takes one input parameter - 1=Onboard, 2=MTMount.
The change in the telemetry source triggers the event ``evt_elevationSource``.

######################
M2 and Camera Hexapods
######################

The M2 and Camera hexapods have identical control interfaces.
The Camera hexapod has ID=1 and M2 hexapod has ID=2.

What the hexapods do is to position themselves to help achieve optimal image quality.
So our discussion focuses on how we command the hexapods to the desired positions.
Here and in the rest of this tech note, position of the hexapod refers to the position (x,y,z) and orientation (rx, ry, rz) around the pivot point.
Unlike for the mirror systems, whose LUTs have to reside with their own control systems due to safety considerations, the LUTs of the hexapods do not have to be with the MTHexapod CSCs.
We choose to make the hexapod LUTs part of the MTHexapod CSCs for consistency across the AOS.

There are various points (x,y,z) and positions (x,y,z,rx,ry,rz) a user needs to understand when operating a hexapod.

- The mechanical zero position, which is (x=0, y=0, z=0, rx=0, ry=0, rz=0) as determined by the actuator encoder readings.
  When the encoder readings are at their pre-calibrated offset values, the hexapod is at its mechanical zero position.
- The pivot point, which is specified in the configuration of the low-level controller.\ [#label1]_
  The pivot point is fixed relative to the hexapod base plate, i.e., it does not move with the hexapod.
  The location of the pivot point relative to the base plate is described in Sec. 2.2 of LTS-206.
- The LUT-commanded collimated position.
  This is where the LUT predicts collimated position should be, which is the best collimated position we can achieve with open-loop AOS control.
  It aims at putting L1S1 first vertex and M2 vertex roughly at the origins of the Camera CS or M2 CS, defined in reference to M1M3.
- The MTAOS-commanded collimated position.
  This is where the MTAOS control predicts the collimated position should be, which is the best collimated position we can achieve with closed-loop AOS control.
  The MTAOS corrections aim at covering the gap between the LUT-commanded collimated position and the ideal collimated position.

.. [#label1] The pivot position can be overridden with the ``setPivot`` CSC command and also in the GUI. In the longer run, we plan to move these to a configuration file. These will rarely need to be changed. Making them easy to change via CSC command or GUI may have unintended consequences.


The actual target position of a hexapod is the sum of two things -

#. The LUT compensation
#. The user-commanded motion

Due to mechanical imperfections in the telescope structure, it is expected that the optimal collimated position of the hexapod will be a bit different from the hexapod mechanical zero position, defined by the encoders and reported by the hexapod control system and MTHexapod CSC.
This will be initially captured by the ASC and kept track of by MTAOS,
and eventually become part of the constant (:math:`C_0`) term in the elevation LUT.

The hexapod position control has a basic operation mode called the *LUT mode*.
By default, the LUT mode is off, #1 above is set to zero.
This is useful for engineering testing. In this mode, the hexapod should simply move to the user-commanded position (depending on whether it is a move (``cmd_move``) or offset (``cmd_offset``) command, it will be in reference to either (0,0,0,0,0,0) or the current position of the hexapod).
When the LUT mode is on, #1 above is determined by the LUTs.
This is useful for science operations. A science user will not have to deal with or even think about how the LUT works.
The MTHexapod CSC will take care of that in the background.
For example, when a science user wants to take an extra-focal image at 1mm, the command is simply ``cmd_move.set_start(z=1000)``.
The MTHexapod CSC will be contantly adjusting the LUT compensation taking into account changes in the elevation angle and temperature etc.

It is expected that the AOS will be primarily using ``cmd_move`` instead of ``cmd_offset``.
We are not removing ``cmd_offset`` because (1) it has been implemented and tested, and (2) just in case we might need it occasionally.

Below we use an example to demonstrate how these commands are supposed to be used. For simplicity, we assume there is only one degree of freedom which is z displacement; the unit is arbituary; and there is only one LUT which only depends on elevation. When the hexapod was last disabled, it was at position z=3.

#. Once we enter enabled state, the hexapod stays at z=3, LUT mode = off;
#. The user sends command ``cmd_move.set_start(z=5)``. The hexapod moves to z=5;
#. The user turns LUT on using ``cmd_enableLut``, the hexapod starts moving toward z=5+3.1=8.1. The 3.1 is from the LUT and calculated using the elevation angle; From this point on, the hexapod will always be calculating the LUT compensation in the background using the latest elevation input etc and adding that compensation to the user-commanded position.
#. Then the user sends command ``cmd_move.set_start(z=-1)``. The hexapod moves to z=-1+3.1=2.1. Assuming the 3.1 has not changed since the step above.
#. The user sends command ``cmd_move.set_start(z=1)``, elevation is unchanged, the hexapod goes to z=1+3.1 = 4.1.
#. Elevation changes result in LUT compensation to change to 3.2, the hexapod moves to z=4.2.
#. The user issues command ``cmd_offset.set_start(z=-1)``, the hexapod moves to z=4.2-1 = 3.2.
#. The user turns off compensation mode using ``cmd_enableLut``, the hexapod stays at z=3.2.

Note that if step 2 is not in place, the hexapod will not move in step 3.
For the compensation mode to take effect, the hexapod has to see a previous move or offset command,
which is associated with an ``evt_cmdPosition`` event.
The hexapod cannot simply take its current position as a commanded uncompensated position.
Otherwise, if the user disable and enable the hexapod a few times in a row, the hexapod will keep drifting.


If a new ``cmd_move`` or ``cmd_offset`` command is issued while the hexapod is in motion,
i.e., moving from point to point, the hexapod CSC would stop the current motion,
then reset the target position and start a new motion.
If the new command is a ``cmd_move`` command, the new target position is solely determined by the new command.
If the new command is a ``cmd_offset`` command, the new target position is obtained by adding the offset to the old target position.

Given this mechanism, in the above example, if ``cmd_enableLut`` is followed immediately by ``cmd_move.set_start(z=-1)``,
before the move to 8.1 finishes,
the hexapod will stop the move toward 8.1, reset the new target as 2.1, and move toward that.

Note that an alternative could be to enable the LUT mode by specifying Lut=True in the move command.
But it feels odd that a move command is needed for enabling and disabling the LUT compensation mode even when no move is desired.

To take intra and extra focal images with offset of 1mm, all that is needed is to turn on the compensation mode, then alternate between ``cmd_move.set_start(z=1000)`` and ``cmd_move.set_start(z=-1000)``. We don’t use ``cmd_offset.set_start(z=2000)`` and ``cmd_offset.set_start(z=-2000)`` because that puts constraints on the position we start with.

When LUT compensation offsets change, events ``evt_lutOffset`` are published.
The parameters include updated input values variables (elevation angle, azimuth angle, camera rotation angle, and temperature)
and updated offsets.
When the compensation mode changes, events ``evt_lutMode`` are published to confirm the change.
Upon issuing a move or offset command, an event ``evt_cmdPosition`` will be published with the updated user-commanded position.
This is the position as determined by the move or offset commands, without taking into account the LUT.
The sum of ``evt_cmdPosition`` and ``evt_lutOffset`` is expected to match the measured position within position accurary as defined by LTS-206.

The actual hexapod position, as measured using encoders for each strut, is published as telemetry under topic ``tel_application.position``.
The demanded position as telemetry is given in ``tel_application.demand``.
The positions of the individual structs are given in ``tel_actuators.calibrated`` and ``tel_actuators.raw``.

When a move is completed, the hexapod publishes an event ``evt_inPosition``.
When individual actuators reach their positions, those are reported with the event ``evt_actuatorInPosition``.

The hexapods slews with the TMA passively - the hexapod legs are much stiffer than M2.

.. _sec-mtaos:

#####
MTAOS
#####

.. note::

   The command structure of MTAOS is currently in the process of being updated. The updated design is described in `tstn-026 <https://tstn-026.lsst.io/>`__. In this section we describe the command structure after this update.

This primary functionalities of the MTAOS are included in its two libraries

- The wavefront estimation pipeline (WEP) takes input images and produces wavefront measurements. The images can be intra and extra focal images from the wavefront sensors or image pairs by exposing the ComCam or LSSTCam at intra and extra focal positions.
- The optical feedback controller (OFC) takes the wavefront measurements produced by the WEP, and calculates the corrections to be applied to the optical system.

To run WEP, the command is ``cmd_runWep``.
The input parameters to ``cmd_runWep`` are the ``dataId`` s of the images.
If it is wavefront sensor images that are being processed, one ``dataId`` should be given.
For processing defocused ComCam or LSSTCam image pairs, two ``dataId`` s are needed.
The wavefront Zernikes (Z4-Z22) for each wavefront sensor, once calculated, are published with events ``evt_wavefrontError``.
If the wavefront solution does not pass the basic quality checks, it is considered invalid and unfit for further use by OFC, ``evt_wavefrontError.valid`` = 0, otherwise it is set to 1.

To run OFC, the command ``cmd_runOfc`` should be used. (To be implemented).
The output of the OFC, formatted as an array of size 50, is published as event ``evt_degreeOfFreedom``.
The parameters of ``evt_degreeOfFreedom`` include both ``aggregate`` which is the total offset in reference to the LUT forces or positions,
and ``visit``, which is the offset from the last visit.
When any of the the corrections fail the basic quality checks,
the corresponding flag is ``evt_degreeOfFreedom.valid`` = 0. Otherwise it is set to 1.
Only when ``evt_degreeOfFreedom.valid`` = 1,
the events ``evt_m1m3Correction``, ``evt_m2Correction``, ``evt_m2HexapodCorrection``, and ``evt_cameraHexapodCorrection`` publish the ``aggregateDOF`` in the format of forces for the mirrors and position offsets for the hexapods.

Detailed control over the various AOS control strategies, for example, which rows or columns we want to truncate from the sensitivity matrix, are realized by dynamically parsing yaml files. An example is found in `tstn-026 <https://tstn-026.lsst.io/>`__.

For computational efficiency, we need to further break down the pipeline and be able to run certain tasks separately.
Since the AOS source selection doesn't require actual images, and only requires pointing information and a bright-star catalog,
we can get it done while waiting for the exposure to finish.
The command for source selection is ``cmd_selectSources``.
Similarly, when the intra focal images and extra focal images come from separate exposures, we can start preprocessing the first exposure images while we wait for the second exposure to finish, up until the step where we need the intra-extra focal pairs to proceed, which is Curvatur Wavefront Sensing (CWFS).
The command for preprocess the image from one side of the focus is ``cmd_preProcess``.

When all the corrections are ready, the command ``cmd_issueCorrection`` is used to send the commands to M1M3, M2 and the hexapods. This involves taking the ``aggregate`` parameter values from ``evt_m1m3Correction``, ``evt_m2Correction``, ``evt_m2HexapodCorrection``, and ``evt_cameraHexapodCorrection`` and send them out using

- M1M3 - ``cmd_applyForces``
- M2 - ``cmd_applyForces``
- M2 and Camera Hexapod - ``cmd_move``

In the scenario that a set of correction was sent out and applied, but we want to reverse to the previous state, the command ``cmd_rejectCorrection`` is used. Note that this is different from the command ``cmd_resetCorrection`` which zeros out all the MTAOS corrections, i.e., the components are set back to positions and forces solely based on the LUTs.

At the beginning of each night, especially during system commissioning when the LUTs are not good enough, the ASC is needed to help put the optical components into the capturing range of the wavefront sensors.
A closed-loop operation needs to be formed between the MTAOS and the ASC, in which the MTAOS repeatedly commands the ASC to measure the positions of the Camera and M2 relative to M1M3, and commands the two hexapods to move.
The iterations stop when the measured positions of the hexapods match the ideal positions within the measurement accuracy of the ASC.
The command to initiate this iterative process is ``cmd_ascAlign``.
The accumulated motions of the hexapods are kept track of using the ``aggregate`` parameters under the topics ``evt_degreeOfFreedom``,
``evt_m2HexapodCorrection``, and ``evt_cameraHexapodCorrection``.
Note that previous design of the ASC had the ASC as the initiator of this measurement process. We've decided to move this to the MTAOS, because it makes it easier for MTAOS to correctly account for all the aggregated motions.

#######################
Alignment System Client
#######################

The alignment system client (ASC) is used at the beginning of each night to put the optical system into the capturing range of the wavefront sensors.
If our LUTs for the hexapods are good enough, i.e., they capture all the dependencies on environmental variables, we will not need the ASC.
But that wouldn't be the case at the beginning, especially for commissioning.
So we expect the ASC will be a critical tool for aligning the telescope.

The command ``cmd_measureTarget`` is used for measuring the position of the target, where target refers to M2 or the Camera.
The positions measured are published as event ``evt_position``.
The measured positions are in M2 CS and Camera CS, respectively, which are defined using M1M3 as the reference.
(The coordinate systems used are configurable. We will need to make sure they are configured as we expect.)

When MTAOS attempts to align M2 and the Camera using the ASC, it uses the command ``cmd_measureTarget`` to take position measurements.
The process is described at the end of Sec. :ref:`sec-mtaos`.

#################
Updating the LUTs
#################

Once we are able to get the system into converged states, we will have a dataset on the AOS corrections for each visit.
The key data we will be looking at are in ``evt_degreeOfFreedom.aggregate``.
These would be easier to make sense than the force and position commands sent out to the components,
because the bending modes are orthonormal, and the force vectors are calculated using the bending mode coefficients.

The trend analyses are conceptually simple - we just need to correlate ``evt_degreeOfFreedom.aggregate`` with the achieved image quality and environmental variables.
The environmental variables includes elevatin angle, azimuth angle, temperature, camera rotator angle, and more.
Any systematic trend will be absorbed into the corresponding component LUT, so that the LUTs alone will be able to get the system into states that are closer and closer to the optimal optical state.

Updating the LUTs themselves will not be an automatic process.
It requires careful analyses and sign-offs from relevant people with the authorities.
Once the LUT updates are authorized, new configuration files will be created and uploaded to the correponding repos.
Users can switch between different versions of the LUTs by following instructions found in `tstn-017 <https://tstn-017.lsst.io>`__.
Once a particular version of a LUT is demostrated to work better than the default,
it becomes the new default, again with appropriate approval.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
