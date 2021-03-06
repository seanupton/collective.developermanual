=============================
 Member manipulation
=============================

.. admonition :: Description

        How to programmatically create, read, edit and delete 
        site members.

.. contents :: :local:

Introduction
============

In Plone, there are two loosely coupled member related subsystems:

* *Authentication and permission* information (``acl_users`` under site
  root), managed by the Pluggable Authentication System (PAS).

* *Member profile* information, accessible through the ``portal_membership``
  tool.


Getting the logged-in member
============================

Anonymous and logged-in member are exposed via :doc:`IPortalState context helper </misc/context>`.

Example::

    from zope.component import getMultiAdapter

    portal_state = getMultiAdapter((self.context, self.request), name="plone_portal_state")
    if portal_state.anonymous():
        # Return target URL for the site anonymous visitors
        return self.product.getHomepageLink()
    else:
        # Return edit URL for the site members
        return product.absolute_url()

or from a template

.. code-block:: html

    <div tal:define="username context/portal_membership/getAuthenticatedMember/getUserName">
        ...
    </div>

Getting any member
==================

To get a member by username (you must have 'manager' role)::

    member = getToolByName(self.context, 'acl_users').getUserById(username)

To get all usernames::

    memberIds = getToolByName(self.context, 'acl_users').getUserIds()

Getting member information
==========================

Once you have acces to the member object, you can grab basic information about it.

Get the user's name:

.. code-block:: python

    member.getName()
    
Get the user's password (Only hash is available and only for standard plone user objects).

.. code-block:: python
    
    acl_users = getToolByName(context, 'acl_users')
    passwdlist = acl_users.source_users._user_passwords
    userpassword = passwdlist.get(member.id, '')

Also, take a look at a script for exporting Plone 3.0 's memberdata and passwords:

* http://blog.kagesenshi.org/2008/05/exporting-plone30-memberdata-and.html



Iterating all site users
============================

Example::

        buffer = ""

        # Returns list of site usernames
        users = context.acl_users.getUserNames()
        # alternative: get user objects
        #users = context.acl_users.getUsers()

        for user in users:
           print "Got username:" + user

.. note::

        Zope users, such as *admin*, are not included in this list.


Getting all *Members* for a given *Role*
========================================

In this example we use the ``portal_membership`` tool.
We assume that a role called 'Agent' exists and that we already
have the context::

    from Products.CMFCore.utils import getToolByName

    membership_tool = getToolByName(self, 'portal_membership')
    agents = [member for member in membership_tool.listMembers() 
                if member.has_role('Agent')]


Groups
======

Groups are stored as ``PloneGroup`` objects. ``PloneGroup`` is a subclass of
``PloneUser``.  Groups are managed by the ``portal_groups`` tool.

* https://github.com/plone/Products.PlonePAS/tree/master/Products/PlonePAS/plugins/ufactory.py

* https://github.com/plone/Products.PlonePAS/tree/master/Products/PlonePAS/plugins/group.py

Creating a group
----------------

Example::

    groups_tool = getToolByName(context, 'portal_groups')

    group_id = "companies"
    if not group_id in groups_tool.getGroupIds():
        groups_tool.addGroup(group_id)

For more information, see:

* https://github.com/plone/Products.PlonePAS/tree/master/Products/PlonePAS/tests/test_groupstool.py

* https://github.com/plone/Products.PlonePAS/tree/master/Products/PlonePAS/plugins/group.py

Add local roles to a group
--------------------------

Example::

   from AccessControl.interfaces import IRoleManager
   if IRoleManager.providedBy(context):
       context.manage_addLocalRoles(groupid, ['Manager',])

.. Note: This is an example of code in a *view*, where ``context`` is
   available.

Update properties for a group
-----------------------------

The ``editGroup`` method modifies the title and description in the
``source_groups`` plugin, and subsequently calls ``setGroupProperties(kw)``
which sets the properties on the ``mutable_properties`` plugin.

Example::

    portal_groups.editGroup(groupid, **properties)
    portal_groups.editGroup(groupid, roles = ['Manager',])
    portal_groups.editGroup(groupid, title = u'my group title')

Getting available groups
------------------------

Getting all groups on the site is possible through ``acl_users`` and the
``source_groups`` plugin, which provides the functionality to manipulate
Plone groups.

Example to get only ids::

    acl_users = getToolByName(self, 'acl_users')
    groups = acl.source_groups.getGroupIds() # Iterable returning id strings

Example to get full group information::

    site = context.portal_url.getPortalObject()
    users = site.acl_users
    group_list = users.source_groups.getGroups()

    for group in group_list:
        # group is PloneGroup object
        yield (group.getName(), group.title)

Adding a user to a group
------------------------

Example::

    # Add user to group "companies"
    portal_groups = site.portal_groups
    portal_groups.addPrincipalToGroup(member.getUserName(), "companies")

Removing a user from a group
------------------------------

Example::

    portal_groups = site.portal_groups
    portal_groups.removePrincipalFromGroup(member.getUserName(), "companies")

Getting groups for a certain user
---------------------------------

Below is an example of getting groups for the logged-in user (Plone 3 and
earlier)::

    portal.portal_membership.getAuthenticatedMember().getGroups()

In Plone 4 you have to use::

    groups_tool = getToolByName(portal, "portal_groups")
    groups_tool.getGroupsByUserId('admin')


Checking whether a user exists
===============================

Example::

        membership = getToolByName(self, 'portal_membership')
        return membership.getMemberById(id) is None

See also:

* http://svn.zope.org/Products.CMFCore/trunk/Products/CMFCore/RegistrationTool.py?rev=110418&view=auto

.. XXX: Why reference revision 110418 specifically?


Creating users
===============

Use the ``portal_registration`` tool. Example::

    def createCompany(request, site, username, title, email, passwd=None):
        """
        Utility function which performs the actual creation, role and permission magic.

        @param username: Unicode string

        @param title: Fullname of user, unicode string

        @return: Created company content item or None if the creation fails
        """

        # If we use custom member properties they must be intiialized
        # before regtool is called
        prepareMemberProperties(site)

        # portal_registration manages new user creation
        regtool = getToolByName(site, 'portal_registration')

        # Default password to the username
        # ... don't do this on the production server!
        if passwd == None:
            passwd = username

        # Only lowercase allowed
        username = username.lower()

        # Username must be ASCII string
        # or Plone will choke when the user tries to log in
        username = str(username)

        def is_ascii(s):
            for c in s:
                if not ord(c) < 128:
                    return False

            return True

        if not is_ascii(username):
            """ """
            IStatusMessage(request).addStatusMessage(_(u"Username must contain only characters a-z"), "error")
            return None

        # This is minimum required information set
        # to create a working member
        properties = {

            'username' : username,

            # Full name must be always as utf-8 encoded
            'fullname' : title.encode("utf-8"),
            'email' : email,
        }

        try:
            # addMember() returns MemberData object
            member = regtool.addMember(username, passwd, properties=properties)
        except ValueError, e:
            # Give user visual feedback what went wrong
            IStatusMessage(request).addStatusMessage(_(u"Could not create the user:") + unicode(e), "error")
            return None

.. XXX: The is_ascii check above doesn't match the error message.

Batch member creation
-----------------------

* http://plone.org/documentation/kb/batch-adding-users


Email login
===========

* Plone 3 does not allow a dot in the username. 
    * This is available as an add-on; see http://plone.org/products/betahaus.emaillogin

* In Plone 4, it is a default feature.


Custom member creation form: complex example
=============================================

Below is an example of a Grok form which the administrator can use to create
new users. New users will receive special properties and a folder for which
they have ownership access.  The password is set to be the same as the
username.  The user is added to a group named "companies".

Example ``company.py``::

    # -*- coding: utf-8 -*-

    """ Add companies.

        Create user account + associated "home folder" content type
        for a company user.
        User accounts have a special role.

        Note: As writing of this 2010-04, needs
        plone.app.directives trunk version which
        contains unreleased validation decorator
    """

    # Core Zope 2 + Zope 3 + Plone
    from zope.interface import Interface
    from zope import schema
    from five import grok
    from Products.CMFCore.interfaces import ISiteRoot
    from Products.CMFCore.utils import getToolByName
    from Products.CMFCore import permissions
    from Products.statusmessages.interfaces import IStatusMessage

    # Form and validation
    from z3c.form import field
    import z3c.form.button
    from plone.directives import form
    from collective.z3cform.grok.grok import PloneFormWrapper
    import plone.autoform.form

    # Products.validation use some ugly ZService magic which I can't quite comprehend
    from Products.validation import validation

    # Our translation catalog
    from isleofback.app import appMessageFactory as _

    grok.templatedir("templates")

    class ICompanyCreationFormSchema(form.Schema):
        """ Define fields used on the form """

        username = schema.TextLine(title=u"Username")

        company_name = schema.TextLine(title=u"Company name")

        email = schema.TextLine(title=u"Email")


    class CompanyCreationForm(plone.autoform.form.AutoExtensibleForm, form.Form):
        """ Form action controller.

        form.DisplayForm will automatically expose the form
        as a view, no wrapping view creation needed.
        """

        # Form label
        name = _(u"Create Company")

        # Which schema is used by AutoExtensibleForm
        schema = ICompanyCreationFormSchema

        # The form does not care about the context object
        # and should not try to extract field value
        # defaults out of it
        ignoreContext = True

        # This form is available at the site root only
        grok.context(ISiteRoot)


        # z3c.form has a function decorator
        # which turns the function to a form button action handler

        @z3c.form.button.buttonAndHandler(_('Create Company'), name='create')
        def createCompanyAction(self, action):
            """
            """

            data, errors = self.extractData()
            if errors:
                self.status = self.formErrorsMessage
                return

            obj = createCompany(self.request, self.context, data["username"], data["company_name"], data["email"])
            if obj is not None:
                # mark only as finished if we get the new object
                IStatusMessage(self.request).addStatusMessage(_(u"Company created"), "info")


    class CompanyCreationView(PloneFormWrapper):
        """ View which exposes form as URL """

        form = CompanyCreationForm

        # Set up security barrier -
        # non-priviledged users can't access this form
        grok.require("cmf.ManagePortal")

        # Use http://yourhost/@@create_company URL to access this form
        grok.name("create_company")

        # This view is available at the site root only
        grok.context(ISiteRoot)

        # Which template is used to decorate the form
        # -> forms.pt in template directory
        grok.template("form")


    @form.validator(field=ICompanyCreationFormSchema['email'])
    def validateEmail(value):
        """
        Use old Products.validation validators to perform the validation.
        """
        validator_function = validation.validatorFor('isEmail')
        if not validator_function(value):
            raise schema.ValidationError(u"Entered email address is not good:" + value)


    def prepareMemberProperties(site):
        """ Adjust site for custom member properties """

        # Need to use ancient Z2 property sheet API here...
        portal_memberdata = getToolByName(site, "portal_memberdata")

        # When new member is created, it's MemberData
        # is populated with the values from portal_memberdata property sheet,
        # so value="" will be the default value for users' home_folder_uid
        # member property
        if not portal_memberdata.hasProperty("home_folder_uid"):
            portal_memberdata.manage_addProperty(id="home_folder_uid", value="", type="string")


        # Create a group "companies" where newly created members will be added
        acl_users = site.acl_users
        #groups = acl_users.source_groups.getGroupIds()
        gr = site.portal_groups

        group_id = "companies"
        if not group_id in gr.getGroupIds():
            gr.addGroup(group_id, [], [],
                        { 'title': 'Companies'})

    def createCompany(request, site, username, title, email, passwd=None):
        """
        Utility function which performs the actual creation, role and permission magic.

        @param username: Unicode string

        @param title: Fullname of user, unicode string

        @return: Created company content item or None if the creation fails
        """

        # If we use custom member properties
        # they must be intiialized before regtool is called
        prepareMemberProperties(site)

        # portal_registrations manages new user creation
        regtool = getToolByName(site, 'portal_registration')

        # Default password to the username
        # ... don't do this on the production server!
        if passwd == None:
            passwd = username

        # Only lowercase allowed
        username = username.lower()

        # Username must be ASCII string
        # or Plone will choke when the user tries to log in
        username = str(username)

        def is_ascii(s):
            for c in s:
                if not ord(c) < 128:
                    return False

            return True

        if not is_ascii(username):
            """ """
            IStatusMessage(request).addStatusMessage(_(u"Username must contain only characters a-z"), "error")
            return None

        # This is minimum required information set
        # to create a working member
        properties = {

            'username' : username,

            # Full name must be always as utf-8 encoded
            'fullname' : title.encode("utf-8"),
            'email' : email,
        }

        try:
            # addMember() returns MemberData object
            member = regtool.addMember(username, passwd, properties=properties)
        except ValueError, e:
            # Give user visual feedback what went wrong
            IStatusMessage(request).addStatusMessage(_(u"Could not create the user:") + unicode(e), "error")
            return None


        # Add user to group "companies"
        portal_groups = site.portal_groups
        portal_groups.addPrincipalToGroup(member.getUserName(), "companies")

        return createMatchingHomeFolder(request, site, member)

    def createMatchingHomeFolder(request, site, member, target_folder="yritykset", target_type="IsleofbackCompany", language="fi"):
        """ Creates a folder, sets its ownership for the member and stores the folder UID in the member data.

        @param member: MemberData object

        @param target_folder: Under which folder a new content item is created

        @param language: Initial two language code of the item
        """

        parent_folder = site.restrictedTraverse(target_folder)

        # Cannot add custom memberdata properties unless explicitly declared

        id = member.getUserName()

        parent_folder.invokeFactory(target_type, id)

        home_folder = parent_folder[id]
        name = member.getProperty("fullname")

        home_folder.setTitle(name)
        home_folder.setLanguage(language)

        email = member.getProperty("email")
        home_folder.setEmail(email)

        # Unset the Archetypes object creation flag
        home_folder.processForm()

        # Store UID of the created folder in memberdata so we can
        # look it up later to e.g. generate the link to the member folder
        member.setMemberProperties(mapping={"home_folder_uid": home_folder.UID()})

        # Get the user handle from member data object
        user = member.getUser()
        username = user.getUserName()

        home_folder.manage_setLocalRoles(username, ["Owner",])
        home_folder.reindexObjectSecurity()


        return home_folder
