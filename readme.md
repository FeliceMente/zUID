## Installation

### Installation prerequisite
If you plan on using the supplied assembly and link job, you will need to customize the DFHEITAL proc from the CICS
SDFHPROC library. For further information regarding this proc, see...
http://www.ibm.com/support/knowledgecenter/en/SSGMCP_5.3.0/com.ibm.cics.ts.applicationprogramming.doc/topics/dfhp3_installprog_cicsproc.html

### Security ###
The default CICS userid will need access to run each instance of the transaction ID. It is recommended a block of
transactions be reserved for the service. The supplied definitions assume zUID transaction will begin with "ID" giving a
range of ID00 to IDZZ.

For those implementations using https and a TCPIPService definition with the AUTHENTICATE parameter set to BASIC, the
authenticated userid will also need access to the transaction.

Administrators should have access to transaction UPLT to define the named counter prior to the first execution.

### Network Considerations ###
The strategy used by this implementation uses a cluster of application owner regions (AORs). No webservice owning
regions (WORs) are employed to route requests over to the AORs. It leverages a combination of sysplex distributor and
port sharing.

The port or ports reserved for this service are defined to a virtual IP address (VIPA) distribute statement (VIPADIST)
and the port is defined as a shared port (SHAREP). Binding the port to the distributed VIPA is optional. With this
arrangement, CICS regons are free to move around and supports more than one region on a LPAR.

The preferred approach is to use a unique host name per instance which will allow a single instance to be moved without
affecting any other instances. The unique host names would be assigned to the VIPA distribute address assigned to the
port(s). An example of unique host name would be zuid01.enterprise-services.mycompany.com matching instance ID01.

The drawback for using a unique host name per instance is certificate handling for https implementations. Either a
wildcard SSL certificate needs to be issued or the SSL certificate needs to have every host name in the subject
alternate name. Using the example above, the wildcard certificate would need to match on
*.enterprise-services.mycompany.com. For the latter scenario, an automated process to add the instance host name to
the subject altername name would be needed to streamline the process.

To provide redundancy across sysplexes a router would be needed to send requests to both sysplexes. Implementing this
level of redundancy requires additional host names to be utilized. There would be the primary host name the application
would use. This host name would be routed across the host name assigned to each sysplex. Using the example above,
the host name supplied to the application would be zuid01.enterprise-services.mycompany.com which would be configured
to be picked up by the router. The router would then choose a sysplex host name to use which could look something like
zuid01.sysplex01.enterprise-services.mycompany.com and zuid01.sysplex02.enterprise-services.mycompany.com. These host
names would be the ones pointing to the VIPA distribute address on their respective sysplex.

### Installation instructions
1. Download the zUID repository to your local workstation.

1. Allocate a JCL and source library on the mainframe. Both libraries will
need to have a record format of FB, a logical record length of 80 and be a dataset type of PDS or PDSE.

1. FTP the JCL in the CNTL folder to the JCL library you have allocated.

1. FTP the source code and definition in the source folder to the source library you have allocated.

1. *In the source library, locate the CONFIG member and edit it.* This file contains a list of configuration items used
to configure the JCL and source to help match your installation standards. The file itself provides a brief
description of each configuration item. Comments are denoted by leading asterisk in the first word. The first word is
the configuration item and the second word is its value.

    1. **@auth@** is the value of the AUTHENTICATE parameter for the https TCPIPService definition. The values can be
    NO, ASSERTED, AUTOMATIC, AUTOREGISTER, BASIC, CERTIFICATE.

    1. **@certficate@** is for CERTIFICATE parameter in the TCPIPService definition for https. Specify certificate as
    CERTIFICATE(server-ssl-certificate-name).

    1. **@cics_csd@** is the dataset name of the CICS system definition (CSD) file.

    1. **@cics_hlq@** is the high level qualifier for CICS datasets.

    1. **@csd_list@** is the CSD group list name. This is the list name to use for the ZUID group.

    1. **@http_port@** is the http port number to be used for ZUID.

    1. **@https_port@** is the https port number to be used for ZUID.

    1. **@job_parms@** are the parameters following the JOB in the JOB card. Be mindful. This substitution will only
    handle one line worth of JOB parameters when customizing jobs.

    1. **@proc_lib@** (Optional) is the dataset containing the customized version of the DFHEITAL proc supplied by IBM.
    If you plan to use the supplied assembly job, the proc library is required.

    1. **@program_lib@** (Optional) is the dataset to be used for zUID programs. If you plan to use the supplied assembly
    job, the program load library is required.

    1. **@source_lib@** is the dataset containing zUID source code.

    1. **@tdq@** is the transient data queue (TDQ) for error messages. Must be 4 bytes.

1. Exit and save the CONFIG member in the source library.

1. In the JCL library, locate and edit the CONFIG member. This is the job that will customize the JCL and source
libraries. Because this job performs the customization, it will need to be customized in order to run. The following
customizations will need to be made.

    1. Modify JOB card to meet your system installation standards.

    1. Change all occurrences of the following.
        1. **@source_lib@** to the source library dataset name. Example. C ALL @source_lib@ CICSTS.ZUID.SOURCE
        1. **@jcl_lib@** to this JCL library dataset name. Example. C ALL @jcl_lib@ CICSTS.ZUID.CNTL

1. Submit the CONFIG job. It should complete with return code 0. The remaining jobs and CSD definitions have been
customized.

1. Assemble the source (Optional). An assembly and link job has been provided to assemble the programs used for ZUID.
    1. Using ASMZUID. The provided job ASMZUID utilizes the DFHEITAL proc from IBM for tranlating CICS commands. The
    DFHEITAL proc must be customized and available in the library specified earlier in the @proc_lib@ configuration item.
    Submit the ASMZUID job. It should end with return code 0.
    1. Using your own assembly and link jobs. If you wish to use your own assembly jobs, here is a list of programs and
    which require the CICS translator.
        1. ZUIDPLT (requires CICS translator)
        1. ZUIDSTCK
        1. ZUID001 (requires CICS translator)

1. Define the CICS resource definitions for zUID. In the JCL library, submit the CSDZUID member. This will install the
minimum number of definitions for ZUID.

1. Define the http port for zUID (optional). If you plan to use an existing http port (TCPIPService definition), you do
not need to submit this job. If you would like to setup a http port specifically for zUID. Submit the CSDZUIDN member in
the JCL library.

1. Define the https port for zUID (optional). The data provided by zUID is generally not regarded as sensitive data as it
is a random string with no relationship. You may however wish to allocate a https port if you intend web browsers to be
able to invoke the service. A https web page will not be able to call a http service directly. If you would like to setup
a https port specifically for zUID. Submit the CSDZUIDS member in the JCL library.

1. Install the ZUID CSD group. No job has been provided to install the ZUID group on the CSD. How CSD groups are
installed varies wildly from company to company. Cold starting CICS, using CICS Explorer, or using CEDA INSTALL are just
some of the ways to perform this action. If cold starting, you may want to hold off until the ZUIDPLT program is ready
to be picked up as entry in the PLTPI in the next few steps.

1. Define program ZUIDPLT as an entry to the PLTPI table in the regions destined to run the zUID services. Refer to
instructions for PLT-program list table in IBM Knowledge Center for CICS.

1. Invoke the ZUIDPLT program. This can be done by restarting the CICS region or by running the UPLT transaction in one
of the CICS regions.

1. Define an instance of ZUID. In the JCL library, the CSDID@@ member provides the JCL to define one instance of zUID.
While some parts of the job are customized, some parameters are left untouched so the process on installing a zUID
instance is repeatable. Keep in mind, you will want some method of keeping track of your clients and which instance of
zUID you have created for them. Recording the path as well is a good idea. Edit CSDID@@ in the JCL library and
customize the following fields.
    1. **@grp_list@** is the CSD group list you wish this instance to be installed.
    1. **@path@** is the path of the zUID instance in the URIMAP definition. It is recommended you prefix the path for all
    instance of zUID to identify the type of service. In the supplied example, the path is prefixed with rzressUID. You
    may want addtional attributes in the path to further organize the services. The last part of the path is generally the
    application name.
    1. **@tran@** is the transaction ID to use for the service. Each instance should get its own transaction ID to provide
    metering capability in relation to the client and their application. The transaction ID is also used as the CSD group
    name by default. This is to aid portability should you decide to move the service.

1. Submit the CSDID@@ job to define the instance.

1. Install the ZUID CSD group. The CSD group name is the transaction ID. No job has been supplied to install the
definitions. Install by cold starting CICS, using CICS Explorer, or using CEDA INSTALL.
