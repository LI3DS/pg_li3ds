########
pg-li3ds
########

|unix_build| |license|

PostgreSQL extension for managing 3D sensor data.


.. contents::

=======
Context
=======

The goal of this project is to manage raw 3D sensor data: lidar pointclouds, trajectories and image metadata with a PostgreSQL database using PostGIS/pointcloud extensions within the context of heterogeneous remote sensing data acquisitions (satellite, aerial, UAV, Mobile mapping system, handheld...).

Three data types are to be used :

- Trajectories
- Lidar point clouds
- Image metadata

The trajectory, lidar and image data is not directly stored in the database. The original data file remain on the file system, and links are created from the database to files on the file system. In the case of the trajectory and lidar data Foreign Data Wrappers are used.

============
Trajectories
============

Trajectories are acquired by geopositioning units fusing a combination of odometer, IMU, gyroscope and GPS sensors. They can also be computed after the fact using photogrammetric such as Structure from Motion (SFM) or lidar pose estimation.
all in all, a trajectory is a series of (time,3D position,3D rotation) samples ordered according to time, which may be interpolated to get the rigid transform (3D position,3D rotation) at any given point in time that transforms the moving trajectory coordinate system to a reference coordinate system in which the trajectory is defined. To enable easy rotation interpolation, quaternion will be used as linear interpolation of quaternions do what is expected when interpolating rotations.
As such, a trajectory will be stored as time ordered and disjoint PCPATCHes `traj` with the following fields:

- ``t`` : time
- ``x,y,z`` : position
- ``qx,qy,qz,qw`` : normalized quaternion

To get the interpolated rigid transform at a given time ``t=1234``, we use ``PC_Interpolate(traj,'t',1234)``.

============
Point clouds
============

------------------
Local Point Clouds
------------------

Lidar sensors produce angular and distance readings within their moving sensor frame. For now we consider lidar point clouds in Cartesian or Spherical coordinates, stored into PCPATCHes `lidar` :

- ``t`` : time
- ``x,y,z`` : Cartesian coordinates
- or ``range,theta,phi`` : Spherical coordinates

To be able to express the local point cloud in a fixed reference frame in the srid of the trajectory, we use ``PC_Interpolate(traj,lidar,'t')``. If the time interval of a `lidar` patch is fully contained within the time interval of a `traj` patch, `PC_Interpolate(traj,lidar,'t')` provides a new trajectory patch with trajectory samples at the same instants as the lidar points (with an unnormalize quaternion, but normalization will be tackled later).

In the general case, we have a column of `traj` patches (with strictly increasing time values) and a column of `lidar` patches (with non-strictly increasing time values, due to multi-echo sensors). The matching of patches will be carried out using the patch min and max time values.

------------------------------------------------------
Multi-echo lidar: Separating Pulse and Echo attributes
------------------------------------------------------

Some of the lidar attributes like ``t,theta,phi,num_echoes`` are shared among all the lidar **echo** samples that were backscattered from the same emitted lidar *pulse*, whereas other attributes like `range,echo,reflectance` are echo-level attributes. `echo` refer to the ordering of the echoes within a pulse composed of ``num_echoes`` echoes.
Storing the local point cloud without separating these attributes into two table is both lossy and suboptimal:

- lossy: we lose the pulse attributes of pulses that did not gather any echo
- sub-optimal: pulse attributes would be repeated for each echo it gathered. Even with dimensional RLE encoding, we are sub-optimal as the
  encoding of the run lengths is currently not shared among dimensions.

Therefore, we are planning to store the echo and pulse attributes as separate PCPATCH columns. The link between pulses and echoes will be performed by storing the partial sum of the ``num_echoes``  attributes as a pulse attribute which acts as a foreign key to the echoes.

-------------------
Sensor calibrations
-------------------

Image, lidar and positionning sensor geometries are described by intrinsic and extrinsic calibrations in the form of  transformation functions between their respective sensor frames (affine transforms, perspective transforms, translations, rotations, scalings, etc with known parameters). The interpolate trajectory is an example of such a transform (a rigid transform in this case, composed of a rotation and a translation).

============
Installation
============

Install postgresql and plpython (The command and package name may have to be adapted for your system) :

.. code-block:: bash

    apt-get install postgresql-plpython-9.6

Create a sample and the required extensions

.. code-block:: bash

    createdb sample
    psql -d sample

.. code-block:: sql

    create extension plpython2u;
    create extension postgis;
    create extension pointcloud;
    create extension pointcloud_postgis;

Install the ``li3ds`` extension and load it into your database::

    git clone https://github.com/li3ds/pg-li3ds
    cd pg-li3ds
    make install
    psql -d sample
    create extension li3ds;

Data model preview:

.. image:: https://cdn.rawgit.com/li3ds/pg-li3ds/master/datamodel.svg
   :target: https://cdn.rawgit.com/li3ds/pg-li3ds/master/datamodel.svg


=========
Run tests
=========

see `tests/readme`_

.. _`tests/readme`: https://github.com/LI3DS/pg-li3ds/blob/master/tests/readme.rst

.. |unix_build| image:: https://img.shields.io/travis/LI3DS/pg-li3ds/master.svg?style=flat-square&label=unix%20build
    :target: http://travis-ci.org/LI3DS/pg-li3ds
    :alt: Build status of the master branch

.. |license| image:: https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square
    :target: https://raw.githubusercontent.com/LI3DS/pg-li3ds/master/LICENSE.txt
    :alt: Package license
