OpenShift Project Enhancement Proposal (PEP)
============================================

Project Enhancement Proposals track technical changes to the OpenShift product.  A PEP is a description of a large technical change in OpenShift that allows developers to understand what the change is, business members to be able to describe where we are going, and product management to create / prioritize stories based on that output.  This is a synthesized, concrete deliverable of an Arch board discussion.

WARNING: All content in this repository is public.  Do not create or submit PEPs that are sensitive in nature or confidential

Current
-------

* [Administrative Console](openshift-pep-001.md)
* [V2 Cartridge Model](openshift-pep-002.md)
* [Asynchronous management tasks](openshift-pep-003.md)
* [Multi-Geography Support](openshift-pep-004.md)
* [Highly Available Web Applications](openshift-pep-005.md)
* [Zero-Downtime Deployment](openshift-pep-006.md)
* [OpenStack Integration](openshift-pep-007.md)

Process
-------

PEPs are currently created out of Architecture Board topics.  Topics are brought up in the Arch Board discussions due to input from developers, users, customers, project managers, competitors, and the community.  Topics which require significant change are discussed until a general approach is agreed on.  With agreement on that approach in hand, a PEP Author will draft a PEP in the correct format and add it to this repository.  

The PEP will be in the "draft" status, indicating it is suitable for general discussion.  The Author will communicate the PEP via the public dev@lists.openshift.redhat.com list.  When posting to the public list the full contents of the PEP should be included, without links.  When sending this email, be sure to include the title and the following prefix in the subject line:

    [openshift-pep] Draft PEP <number> <title>

Review comments from stakeholders and the community will be integrated into the PEP and should be conducted on the mailing lists.

After sufficient review and feedback (at the Author's discretion), PEPs will be moved to the "reviewed" state.  A PEP in this state will be discussed at an Arch Board meeting and, barring significant disagreement, marked "accepted".  PEPs will remain in the accepted state until an implementation is available in the Origin master branch at https://github.com/openshift/origin-server.


Participation
-------------

If there is a significant architectural change that you feel belongs in OpenShift, please discuss it on dev@lists.openshift.redhat.com.  If there is sufficient consensus on the change to warrant a PEP, please describe in detail the elements that comprise the PEP in an email to the list.

Template for new PEPs: [template.md](template.md)
