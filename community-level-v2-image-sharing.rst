=================================
Add community-level image sharing
=================================

https://blueprints.launchpad.net/glance/+spec/community-level-v2-image-sharing

Add a new feature which allows a user to share an image with all other tenants.
These "community" images do no appear in image listings for a user until that
user has accepted the image.


Problem description
===================

Currently, images must either be public, private or shared. There is no
provision for an image to be available for use by users other than the owner of
the image without making the image either public or explicitly shared by the
owner of the image with other users. If the number of users who require access
to the image is large, the overhead of explicit sharing can become a burden on
the owner of the image.


Use cases
---------

1. An open source project wants to make a prepared VM image available so users simply boot an instance
   from that image , to use the project's software. The developers of the project are too busy writing software to worry about the
   hassle of maintaining a list of "customers" as they'd have to do with
   current v2 image sharing. At the same time, the cloud provider hosting the
   prepared image doesn't want to make this image public, as that would imply
   a support relation that doesn't exist between image consumers and the cloud
   provider.

2. A cloud provider wants to discourage users from using a public image that
   is getting old, for example an image which is missing some security patches
   or is no longer supported by the vendor, but doesn't want to delete the image because
   some users may still require it. Users could need the old immage to rebuild a server, or because they have custom patches for
   that, particular image. 
   
   In either case, the vendor could make the image a "community' image and alieviate the challenges presented above. This means that:

   a) The community image won't appear in user's default image lists. This means they won't
   know about it unless they are motivated to seek it out, for example by
   asking other users for the UUID.

   b) Since the image is no longer "public," it doesn't imply the same
   level of support from the vendor as a provider-supplied, public image.


Proposed change
===============

The chosen method of implementing this functionality is to add a membership
record for an image that has a target of ``"*"`` (i.e. it is shared with all
tenants) but with ``membership_status = "community"``. This marks it as a
community image very simply and requires few modifications to existing code.

This respects the current anti-spam provisions in the glance v2 api; when an
image owner makes an image a "community" image, any other tenant should be
able to boot an instance from that image, but the image will not show up in any
tenant's default image-list.


Alternatives
------------

Adding image aliases
~~~~~~~~~~~~~~~~~~~~

A completely different way of solving this usecase for cloud providers
(discouraging users from using an older version of a public image) could be to
create a mechanism to make an image alias, which could point at the newest
version of the public image. There is a abandoned blueprint for this feature
[#]_. This, however, is much harder to implement and does not fit with the
other usecases.

.. [#] https://blueprints.launchpad.net/glance/+spec/glance-image-aliases


Data model impact
-----------------

None

REST API impact
---------------

Image discovery
~~~~~~~~~~~~~~~

Community images will be displayed in an image listing only if the
``visibility`` filter is set to the value ``'community'``. ::

    GET /v2/images?visibility=community


All other appropriate filters will be respected. Of note is the use of an ``owner``
parameter. This, when supplied together with the ``community`` filter, allows a
user to request only those community images owned by that particular tenant: ::

    GET /v2/images?visibility=community&owner={tenantId}


Accepting a community Image
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Having to filter the results each time to find a particular image is
inconvenient, particularity if that image is often used. To remove this
inconvenience, users are allowed to "bookmark" a particular community image.
This image then appears in the user's image-list. This functionality operates
in much the same way that v2 image sharing works, but the owner of the image
does not have to maintain relationships with each of the new image consumers.

If a consumer of a community image would like to bookmark it, they must:

1. Add their tenant ID as a member of the community image.

2. Accept the image so it will appear in their image-list


For this, the existing image-sharing API calls will be used:

1. Add the tenant ID to the image: ::

       POST /v2/images/{image_id}/members

   Request body: ::

       {"member": "8989447062e04a818baf9e073fd04fa7"}

   The response and other behaviour remains the same as was previously defined
   for this call.

2. Accept the image: ::

       PUT /v2/images/{image_id}/members/{member_id}

   Request body: ::

       {"status": "accepted"}

   Again, the response and other behaviour remains the same as was previously
   defined for this call.


Making an image a "community image"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The owner of an image can use the existing image-update call, changing the
image's visibility to ``'community'``: ::

    PATCH /v2/images/{image_id}

Request body:

.. code:: json

    [{ "op": "replace", "path": "/visibility", "value": "community" }]

The response and other behaviour remains the same as was previously defined for
this call.


Removing a community image
~~~~~~~~~~~~~~~~~~~~~~~~~~

A community image can be removed from community-level access by also using the
image-update call. Instead of setting it to ``'community'`` as before, we set
it to ``'private'``: ::

    PATCH /v2/images/{image_id}

Request body:

.. code:: json

    [{ "op": "replace", "path": "/visibility", "value": "private" }]

As in all the above cases, the response and other behaviour remains the same as
was previously defined for this call.


Security impact
---------------

Little to none. The only risk is for users to accidentally leak potentially
sensitive images by accidentally making them community images.

Notifications impact
--------------------

None

Other end user impact
---------------------

Python-glanceclient will be updated to expose this feature. An option to
``glance image-update`` will be added named ``--visibility
<VISIBILITY_STATUS>``, where ``VISIBILITY_STATUS`` may be one of ``{public,
private, community}``.

For example, to make an image a community image:

.. code:: bash

    $ glance image-update --visibility community <IMAGE>

To make the image private again:

.. code:: bash

    $ glance image-update --visibility private <IMAGE>


Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kragniz

Other contributors:
  iccha-sethi

Work Items
----------

- Add functionality for storing the community state in the interfaces to both db
  backends:

  + sqlalchemy

  + simple

- Add functionality to enable this and accepting the image in the api

- Add unit tests to test various inputs to the api

- Add functional tests for the lifecycle of community images

- Update glanceclient with the new option


Dependencies
============

None

Testing
=======

A tempest test must be added to cover creating a community image and it
transitioning between public and private states.


Documentation Impact
====================

New features must be documented in both glance and python-glanceclient.

References
==========

None
