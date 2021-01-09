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

The M1M3 here refers to the M1M3 support system only, not including the M1M3 thermal system (https://github.com/lsst-ts/ts_M1M3Thermal).
The latter is also important for the system image quality, but does not directly interact with the AOS, nor the support system.
The same applies to the camera rotator (https://github.com/lsst-ts/ts_mtrotator), and the
Vibration Monitoring System (VMS https://github.com/lsst-ts/ts_vms).
The VMS has components on M1M3, M2, the camera rotator, and the Telescope Mount Assembly (TMA).

The command structure and the naming of the variables are defined by the Interface Control Documents (ICDs): LTS-161 (M1M3), LTS-162 (M2), LTS-160 (hexapods) and MTAOS (LTS-163??).
Their implementations are found in the XML:

- M1M3 https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTM1M3
- M2 https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTM2
- M2 and Camera Hexapods https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/Hexapod
- MTAOS https://github.com/lsst-ts/ts_xml/tree/develop/sal_interfaces/MTAOS

Or, in more human-readable format:

- M1M3 https://ts-xml.lsst.io/sal_interfaces/MTM1M3.html
- M2 https://ts-xml.lsst.io/sal_interfaces/MTM2.html
- M2 and Camera Hexapods https://ts-xml.lsst.io/sal_interfaces/MTHexapod.html
- MTAOS https://ts-xml.lsst.io/sal_interfaces/MTAOS.html

Each CSC also has its own documentation page:

- M1M3 https://github.com/lsst-ts/ts_m1m3support/blob/master/README.md
- M2 https://github.com/lsst-ts/ts_mtm2/blob/master/README.md
- M2 and Camera Hexapods https://ts-mthexapod.lsst.io
- MTAOS https://ts-mtaos.lsst.io

.. MTAOS architecture and design considerations are also discussed in `tstn-026 <https://tstn-026.lsst.io/>`__.

This purpose of this tech note is to describe the logic behind the design, and how the commands and variables are intended to be used.
We only focus on the force related commands and variables for M1M3 and M2, and position related commands and variables for the hexapods.
There are a lot more commands and varialbes for each CSC that are not covered here.

####
M1M3
####

On the top level, the M1M3 actuator forces, including x, y, and z-forces, fall into four categories.

#. Active optic forces. The command to apply the forces is ``cmd_applyActiveOpticForces``. The input to the command is an 1D array of size 156. Each element of the array is a z-force values to be applied to a specific actuator. The actuators are ordered with ascending IDs, as found `here <https://github.com/lsst-ts/ts_m1m3supportgui/blob/develop/FATABLE.py/>`__. Every time the active optic forces are applied, there is an event ``evt_appliedActiveOpticForces`` which publishes the forces being applied. The input forces are aggregated AOS forces, instead of additional AOS forces. Therefore, a new ``cmd_applyActiveOpticForces`` command overrides all previous ``cmd_applyActiveOpticForces`` commands.

#. Force-balance (FB) forces. Once the FB system is enabled (using command ``cmd_enableHardpointCorrections``), the FB forces are automatically applied based on the net forces and moments measured by the hard points. When the FB forces change, events ``evt_appliedBalanceForces`` are published. The published parameters include the 12 x-forces, 100 y-forces, 156 z-forces, and the x, y, and z net forces and moments.


#. Slewing forces. The forces needed for slewing the M1M3 together with the TMA are applied automatically by the control system. Therefore there is no separate command for applying these forces. These forces are calculated based on measurements from onboard accelerometers (not the VMS accelerometers) and gyros. They are published as events ``evt_appliedAccelerationForces`` and ``evt_appliedVelocityForces``.

#. Look-Up Table (LUT) forces. The M1M3 LUT forces have the following components:

  - Elevation dependency. The corresponding force component is ``evt_appliedElevationForces``. Note that the LUT component due to Force actuator internal weights is included here, because it is elevation dependent. See `Sec. 5 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#actuator-internal-weight-components/>`__ for more details.
  - Azimuth dependency. The corresponding force component is ``evt_appliedAzimuthForces``.
  - Temperature dependency. The corresponding force component is ``evt_appliedThermalForces``.
  - Static bending forces. The corresponding force component is ``evt_appliedStaticForces``.
  - Load cell zero offset. To be implemented. See `Sec. 10 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#load-cell-zero-offsets/>`__.

The total applied forces, which is the sum of the above categories, are published as ``evt_appliedForces``.
The events ``evt_appliedCylinderForces`` give the same information, but all the forces are converted into cylinder forces instead of x, y, and z-forces.

The forces, as measured by the load cells, are given as telemetry under the topic ``tel_forceActuatorData``.
Under this topic, the user can find 156 ``primaryCylinderForces``, 112 ``secondaryCylinderForces``, 12 ``xForces``, 100 ``yForces``, 156 ``zForces``, and the net x, y, z-forces and x, y, z-moments.

There are two additional commands for applying forces: ``cmd_applyAberrationForces`` and ``cmd_applyOffsetForces``.
``cmd_applyAberrationForces`` is used for manually injecting aberrations during AOS tests. The forces added by this command is kept track of by the event ``evt_appliedAberrationForces``.
The command ``cmd_applyOffsetForces`` is used for manually adding additional forces to individual actuators, which can be used during bump test, for example.
The corresponding event is ``evt_appliedOffsetForces``.

The M1M3 LUT forces are on whenever the mirror control system is in the enabled state.
Due to delay in TMA telemetry, M1M3 uses its own inclinometer reading as input to the elevation LUT.
Only when the elevation angle from the onboard inclinometer is not available would the TMA telemetry be used.
Similarly, the determination of the slewing forces uses the onboard accelerometers and gyro by default.
It only falls back to the TMA telemetry when due to some reason the readings from the onboard instruments are not available.

##
M2
##

Similarly, for M2, the total forces could have the following components -

#. Acitve optic forces. The command for applying active optic forces is ``cmd_applyForces``. The command takes two 1D arrays as inputs, one for axial forces, with size 72, and the other for tangent forces, with size 6. The ordering of the axial actuators goes as B1, B2, ..., B30, C1, C2, ..., C24, D1, D2, ..., D18. For the tangent links it is A1, A2, ..., A6. The forces applied by this command are kept track of as telemetry under ``tel_axialForce.applied`` and ``tel_tangentForce.applied``, containing 72 axial forces and 6 tangent forces, respectively.  The input forces are aggregated AOS forces, instead of additional AOS forces. Therefore, a new ``cmd_applyForces`` overrides all previous ``cmd_applyForces`` commands.

#. Force-balance (FB) forces. The FB system is on by default, but can be switched on/off using command ``cmd_switchForceBalanceSystem``, which triggers the event ``evt_forceBalanceStatus``. When the FB system is enabled, the FB forces are automatically applied based on the net forces and moments measured by the hard points. The FB forces are kept track of as telemetry under ``tel_axialForce.hardpointCorrection`` and ``tel_tangentForce.hardpointCorrection``, containing 72 axial forces and 6 tangent forces, respectively. The net forces and moments created by the FB forces can be calculated using FB forces on individual actuators. For convenience, they are also given separately under the telemetry topic ``tel_forceBalance``.

#. Look-Up Table (LUT) forces. The M2 LUT forces have the following components:

  - Elevation dependency. The corresponding force component are ``tel_axialForce.lutGravity`` and ``tel_tangentForce.lutGravity``. Note that the LUT component due to force actuator internal weights is included here, because it is elevation dependent. See `Sec. 5 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#actuator-internal-weight-components/>`__ for more details. In Harris' language, the elevation dependent components include FE and FA. See `this notebook <https://github.com/lsst-sitcom/M2_summit_2003/blob/master/a03_LUT_0.ipynb/>`__.
  - Azimuth dependency. The corresponding force component are ``tel_axialForce.lutAzimuth`` and ``tel_tangentForce.lutAzimuth``. (To be implemented)
  - Temperature dependency. The corresponding force component are ``tel_axialForce.lutTemperature`` and ``tel_tangentForce.lutTemperature``.
  - Static bending forces. The corresponding force component is not published in DDS, since they do not change with elevation. In Harris' lauguage, this includes `F0 <https://github.com/lsst-ts/ts_mtm2_cell/blob/master/configuration/lsst-m2/config/parameter_files/luts/FinalOpticalLUTs/F_0.csv/>`__ and `FF <https://github.com/lsst-ts/ts_mtm2_cell/blob/master/configuration/lsst-m2/config/parameter_files/luts/FinalOpticalLUTs/F_F.csv/>`__.
  - Load cell zero offset. To be implemented. See `Sec. 10 of SITCOMTN-004 <https://sitcomtn-004.lsst.io/#load-cell-zero-offsets/>`__.

Note that for M2, the force components are found in telemetry. They are part of the telemetry instead of events mostly due to historical reasons.
Since the amount of data is much smaller than M1M3, we did not make a change to this.

The command ``cmd_applyForces`` is not only intended for closed-loop AOS. It is also supposed to be used for manual AOS test, and bump test, where an arbituary set of forces need to be applied. The forces applied by ``cmd_applyForces`` can be cleared using ``cmd_resetForceOffsets``, which is a shorthand for a new ``cmd_applyForces`` command with all demanded forces set to zero.

There are no forces for slewing, because M2 slews passively. The actuators are much stiffer than M1M3. When the TMA slews, the M2 hard points will sense the load and the FB system will offload the forces and moments.

The forces, as measured by the load cells, are given as telemetry under ``tel_axialForce.measured`` and ``tel_tangentForce.measured``.

The M2 LUT forces are on whenever the mirror control system is in the enabled state.
By default, the elevation angle used by the elevation dependent component of the LUT is from the onboard inclinometer.
The source for the elevation angle can be switched between the onboard inclinometer and the TMA telemetry using the
command ``cmd_selectInclinationTelemetrySource``, which takes one input parameter - 1=Onboard, 2=MTMount.
The change in the telemetry source triggers the event ``evt_inclinationTelemetrySource``.

######################
M2 and Camera Hexapods
######################

The M2 and Camera hexapods have identical control interfaces. The Camera hexapod has ID=1 and M2 hexapod has ID=2.

What the hexapods do is to position themselves to help acheive optimal image quality. So our discussion focuses on how we command the hexapods to the desired positions. Unlike for the mirror systems, whose LUTs have to reside with their own control systems due to safety considerations, the LUTs of the hexapods do not have to be with the hexapod CSCs. We choose to have the hexapod LUTs to be part of their owns CSCs for consistency across the AOS.

The actual target position of a hexapod is the sum of three things -

#. The reference position
#. The LUT compensation
#. The user-commanded motion

Due to hexapod installation, it is expected that the optimal collimated position of the hexapod will be a bit different from the hexapod (0,0,0) defined by the encoders and reported by the hexapod control system and CSC. The reference position is meant to be the approximately collimated position as defined in the hexapod hardware systems. It is configurable by the CSCs. The command ``cmd_moveToReference`` moves the hexapod to this position.
*The concept of the reference position helps decouple the LUT from the hexapod installation.*

The hexapod position control has a basic operation mode called the *compensation mode*.
When the compensation mode is off, #2 above is set to zero.
This is useful for engineering testing. In this mode, the hexapod should simply move to the user-commanded position (depending on whether it is a move (``cmd_move``) or offset (``cmd_offset``) command, it will be in reference to either the reference position or the current position of the hexapod).
When the compensation mode is on, #2 above is determined by the LUTs.
This is useful for science operations. A science user will not have to deal with or even think about how the LUT works.
The hexapod CSC will take care of that in the background.
For example, when a science user wants to take an extra-focal image at 1mm, the command is simply ``cmd_move.set_start(z=1000)``.
The CSC will be contantly adjusting the LUT compensation taking into account changes in the elevation angle and temperature etc.

Note that there are a few *special* spacial points in terms of operating a hexapod.

- The (0,0,0) as determined by the hexapod encoders - All the positions reported by the hexapod control system and CSC are in reference to a coordinate system (CS) with this point as origin.
- The pivot point, as defined by the user using command ``cmd_pivot`` - the input x, y, and z to this command is defined in the CS above. For AOS operations, we will mostly use L1S1 first vertex and M2 vertex as pivot points for the Camera and M2 hexapods, respectively;
- The reference position, which puts L1S1 first vertex and M2 vertex roughly at the origins of the CCS or M2CS, defined in reference to M1M3.

Below we use an example to demonstrate how these commands are supposed to be used. For simplicity, we assume there is only one degree of freedom which is z displacement; the unit is arbituary; and there is only one LUT which only depends on elevation. As part of the CSC configuration we define the reference position of the hexapod, default z=3. When the hexapod was last disabled, it was at position z=-5.

#. Once we enter enabled state, the hexapod stays at z=-5, compensation mode = off;
#. The user turns compensation on using ``cmd_setCompensationMode``, the hexapod moves to z=-5+0.1=-4.9. The 0.1 is calculated using the elevation angle;
#. Then the user sends command ``cmd_move.set_start(z=1)``, elevation stays unchanged. The hexapod moves to z=3+0.1+1=4.1;
#. The user sends command ``cmd_offset.set_start(z=1)``, elevation is unchanged, the hexapod goes to z=5.1.
#. Elevation changes result in LUT compensation to change to 0.2, the hexapod moves to z=5.2.
#. The user issues command ``cmd_moveToReference``, the hexapod moves to z=3+0.2 = 3.2.
#. The user turns off compensation mode using ``cmd_setCompensationMode``, the hexapod moves to z=3.


To take intra and extra focal images with offset of 1mm, all that is needed is to turn on the compensation mode, then alternate between ``cmd_move.set_start(z=1000)`` and ``cmd_move.set_start(z=-1000)``. We don’t use ``cmd_offset.set_start(z=2000)`` and ``cmd_offset.set_start(z=-2000)`` because that puts constraints on the position we start with.

Initially the reference position is set at (0,0,0) as determined by the hexapod encoders.
If due to installation we find that under normal operation conditions the best focus is far away from reference position, that large offset will need to be absorbed into reference position.

When LUT compensation offsets change, events ``evt_compensationOffset`` are published.
The parameters include updated input values variables (elevation angle, azimuth angle, camera rotation angle, and temperature)
and updated offsets.
When the compensation mode changes, events ``evt_compensationMode`` are published to confirm the change.
Upon issuing a move or offset command, an event ``evt_uncompensatedPosition`` will be published with the updated uncompensated position. The uncompensated position is the sum of the reference position and the move command parameters, or the sum of the current position and the offset command parameters.
Adding the LUT compensation offsets to the uncompensated position gives the compensated position.
When there is a change in the compensated position, an event ``evt_compensatedPosition`` is published.

The hexapods can not accept a new ``cmd_move`` or ``cmd_offset`` command before the previous move finishes or the move be stopped by command ``cmd_stop``.
When a move is completed, the hexapod publishes an event ``evt_inPosition``.
When individual actuators reach their positions, those are reported with events ``evt_actuatorInPosition``.

The actual hexapod position, as measured using encoders for each strut, is published as telemetry under topic ``tel_application.position``.
The demanded position as telemetry is given in ``tel_application.demand``.
The positions of the individual structs are given in ``tel_actuators.calibrated`` and ``tel_actuators.raw``.

The hexapods slews with the TMA passively - the hexapod legs are much stiffer than M2.

#####
MTAOS
#####

.. note::

   The command structure of MTAOS is currently in the process of being updated. The updated design is described in `tstn-026 <https://tstn-026.lsst.io/>`__. In this section we describe the command structure after this update.

This primary functionality of the MTAOS is to process wavefront images from either corner wavefront sensors or defocused ComCam or LSSTCam, measure wavefront in the form of Zernike coefficients, and send out corrections to the optical system which are based on the wavefront measurements.

To process corner wavefront sensor images, the command is ``cmd_processWavefrontError``.
To process intra and extra focal image pairs, taken by pistoning either the ComCam or LSSTCam, the command ``cmd_processIntraExtraWavefrontError`` should be used.
Either of these commands runs both the wavefront estimation pipeline (WEP) and the optical feedback controller (OFC).
The wavefront Zernikes (Z4-Z22) for each wavefront sensor, once calculated, are published with events ``evt_wavefrontError``.
If the wavefront solution does not pass the basic quality checks, it is considered invalid and unfit for further use by OFC, an event ``rejectedWavefrontError`` is published.
The output of the OFC, formatted as an array of size 50, is published as event ``evt_degreeOfFreedom``.
The parameters of ``evt_degreeOfFreedom`` include both ``aggregateDOF`` which is the total offset in reference to the LUT forces or positions,
and ``visitDOF``, which is the offset from the last visit.
The events ``evt_m1m3Correction``, ``evt_m2Correction``, ``evt_m2HexapodCorrection``, and ``evt_cameraHexapodCorrection`` publish the ``aggregatedDOF`` in the format of forces for the mirrors and position offsets for the hexapods.
When any of the the corrections fail the basic quality checks,
the corresponding events are ``evt_rejectedDegreeOfFreedom``, ``evt_rejectedM1m3Correction``, ``evt_rejectedM2Correction``, ``evt_rejectedM2HexapodCorrection``, and ``evt_rejectedCameraHexapodCorrection``.

We need to be able to run WEP and OFC separately. There are times we need to run WEP on a lot of images just to analyze the wavefront measurements, without involving OFC. The command ``cmd_runWep`` should be used. (To be implemented).
Other times for debugging and testing purposes we want to add certain wavefront aberrations in the form of Zernikes to the optical system.
So we need to be able to run OFC only. The command ``cmd_runOfc`` should be used. (To be implemented).
Detailed control over the various AOS control strategies, for example, which rows or columns we want to truncate from the sensitivity matrix, are realized via configuration changes.

For computational efficiency, we need to further break down the pipeline and be able to run certain tasks separately.
Since the AOS source selection doesn't require actual images, and only requires pointing information and a bright-star catalog,
we can get it done while waiting for the exposure to finish.
The command for source selection is ``cmd_selectSources``.
Similarly, when the intra focal images and extra focal images come from separate exposures, we can start preprocessing the first exposure images while we wait for the second exposure to finish, up until the step where we need the intra-extra focal pairs to proceed, which is Curvatur Wavefront Sensing (CWFS).
The command for preprocess the image from one side of the focus is ``cmd_preProcess``.

When all the corrections are ready, the command ``cmd_issueWavefrontCorrection`` is used to send the commands to M1M3, M2 and the hexapods. This involves taking the ``aggregateDOF`` parameter values from ``evt_m1m3Correction``, ``evt_m2Correction``, ``evt_m2HexapodCorrection``, and ``evt_cameraHexapodCorrection`` and send them out using

- M1M3 - ``cmd_applyActiveOpticForces``
- M2 - ``cmd_applyForces``
- M2 and Camera Hexapod - ``cmd_move``

In the scenario that a set of correction was sent out and applied, but we want to reverse to the previous state, the command ``cmd_resetWavefrontCorrection`` is used.

#################
Updating the LUTs
#################

Once we are able to get the system into converged states, we will have a dataset on the AOS corrections for each visit.
The key data we will be looking at are in ``evt_degreeOfFreedom.aggregateDOF``.
These would be easier to make sense than the force and position commands sent out to the components,
because the bending modes are orthonormal, and the force vectors are calculated using the bending mode coefficients.

The trend analyses are conceptually simple - we just need to correlate ``evt_degreeOfFreedom.aggregateDOF`` with the achieve image quality and environmental variables.
The environmental variables includes elevatin angle, azimuth angle, temperature, camera rotator angle, and more.
Any systematic trend will be absorbed into the corresponding component LUT, so that the LUTs alone will be able to get the system into states that are closer and closer to the optimal optical state.



.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
