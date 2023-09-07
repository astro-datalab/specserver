> [!IMPORTANT]
> Update 2023-Sept-6: This repository has been archived, and is now read-only. The spectroscopic data service enabled by this code has been retired. Please use the newer SPARCL service from here on: https://astrosparcl.datalab.noirlab.edu
                        
# Spectroscopic Data Service

The spectroscopic data service is meant to provide a high-throughput
data query and access interface for spectral data.  The initial release 
focuses on SDSS DR8 thru DR16 and will be expanded to include future 
spectral surveys such as DESI.  

Performance can be up to 100X faster than similar interfaces for SDSS
data, but is variable depending on the level of processing requested and 
the specific data release.  The SDSS DR16 data are pre-processed to a 
directory of saved 'numpy' arrays that constitute the core FITS BINTABLE
of the data release, however on-the-fly extraction for earlier releases 
makes all FITS data accessible.


## Client Interface

    Common Interface

        client = getClient  (context='<context>', profile='<profile>')

          status = isAlive  (svc_url=DEF_SERVICE_URL, timeout=2)

               set_svc_url  (svc_url)
     svc_url = get_svc_url  ()

               set_context  (context)
         ctx = get_context  ()
      ctxs = list_contexts  (context, fmt='text')
      ctxs = list_contexts  (context=None, fmt='text')

               set_profile  (profile)
        prof = get_profile  ()
     profs = list_profiles  (profile, fmt='text')
     profs = list_profiles  (profile=None, fmt='text')

       catalogs = catalogs  (context='default', profile='default')

    Query Interface

        Query for a list of spectrum IDs that can then be retrieved from
        the service.  Positional queries in the form of polygonal regions,
        an astropy Coord object or explicit position are supported.  The
        method returns a list of unique spectrum object IDs to be accessed.
        Valid ID values are specific to the data context.

        The 'constraint' parameter may be specified as a valid SQL 'where'
        clause to be added to the query (e.g. to do a color cut of objects).
        [Use of this parameter requires knowledge of the schema being used.]

        The DL user auth token is passed automatically in the X-DL-AuthToken
        header, it may be overridden with the 'token' kw parameter.


          id_list = query (<region> | <coord, size> | <ra, dec, size>,
                           constraint=<sql_where_clause>, out='',
                           context=None, profile=None,
                           **kw)
                where:
                    region      Array of polygon vertex tuples (in deg)
                    coord       Astropy Coord object
                    ra, dec     RA/Dec position (in deg)
                    size        Search size (in deg)
                    out         Save query results to filename
                    constraint  A valid SQL 'where' clause
                    context     Dataset context
                    profile     Service profile
                    kw          optional parameters

                returns:
                    An array of object identifiers for the given context. The
                    id types will be specific to the dataset and selected by
                    a 'fields' kw param or from a specific table, e.g. for 

                    kw options:
                      for context='sdss_dr16':
                        fields:
                            specobjid           # or 'bestobjid', etc
                            tuple               # a plate/mjd/fiber/run2d tuple

                            Service will always return array of 'specobjid'
                            value, the p/m/f tuple is extracted from the
                            bitmask value by the client.

                        primary:
                            True                # query sdss_dr16.specobj
                            False               # query sdss_dr16.specobjall
                        catalog:
                            <schema>.<table>    # alternative catalog to query
                                                # e.g. a VAC from earlier DR
                                                # (must support ra/dec search
                                                # and return specobjid-like
                                                # value)

                      for all contexts/profiles:
                        timeout=<timeout>       # query timeout
                        token=<token>           # to pass alternate auth token
                        debug                   # client debug flag
                        verbose                 # client verbosity flag

        Example return values:
            [7201313360318844928, 8170840728492331008, ....]
            [(6396,56358,209), (7257,56658,673), .... ]

        Examples:
            1) Query by polygonal region:

                region = [(0.0,0.0),(1.0,0.0),(0.0,1.0)]
                id_list = spec.query (region)

            2) Query a 0.1deg cone by by astropy Coord:

                from astroquery.sdss import SDSS
                from astropy import coordinates as coords
                pos = coords.SkyCoord('0h8m05.63s +14d50m23.3s', frame='icrs')

                id_list = spec.query (pos, size=0.1)

            3) Query by position:

                id_list = spec.query (0.125, 12.123, 0.1)

            4) Query objects in the DR12 Portsmouth emission line catalog
               is a 10deg cone around (0.0,0.0), return (plate,mjd,fiber):

                id_list = spec.query (0.0, 0.0, 10.0,
                                      catalog='sdss_dr12.emissionlinesport',
                                      fields='tuple')


    Data Access Interface

        The Data Access interface is used to retrieve spectra identified by
        an object ID list.  That list can be generated by the query() method
        above, or any other DL query that can produce a valid list of object
        IDs.  A single-object identifier need not be an array, the type of
        identifier allowed (e.g. a int64 or a tuple) is determined by the
        dataset 'context' parameter.  The method returns an array (or single
        value) of the requested format type with one spectrum array for
        each object in the ID list.

        The 'align' parameter can be enabled to re-grid all spectra to a
        common wavelength scale, zero-padding each array as needed.  The
        starting and ending wavelength of aligned data will be the global
        min/max values for all spectra in the list.  If the 'cutout' parameter
        is enabled, data will be excised to the specified boundaris and padded
        as needed (i.e. a cutout implies align=True).

        All data are returned to the caller as and array of the requested 
        type unless the 'out' parameter specifies a directory location in
        which to store the data.  This may be a unix directory path available
        to the user, or a virtual storage URI (e.g. 'vos://myspec/').  If
        fmt='nunpy' then a NumPy save file is created; if the fmt='FITS"
        then the original data release FITS file of coadd spectra is returned.
        The name of the spectrum file on the server is used automatically.

        The DL user auth token is passed automatically in the X-DL-AuthToken
        header, it may be overridden with the 'token' kw parameter.


            list = getSpec  (id_list, fmt='numpy',
                             out=None, align=False, cutout=None,
                             context=None, profile=None,
                             **kw)
                where:
                    id_list     List of object IDs (dataset-specific).
                                Must be one of:
                                    - single string/int/int64/tuple identifier
                                    - python array/list object
                                    - string of identifier values (one/line)
                                    - filename containing identifiers (one/line)
                                    - VOS name containing identifiers (one/line)
                    fmt         Result format, one of:
                                    numpy
                                    pandas
                                    Spectrum1D
                                    Spectrum1DCollection
                                    Spectrum1DList
                                    FITS
                    out         Output file location (fmt=FITS)
                    align       Align spectra to common wavelength grid
                    cutout      Cutout range as '<start>-<end>'
                    context     Dataset context
                    profile     Service profile
                    kw          optional parameters

                returns:
                    An array of data objects in the requested format, or a
                    single object of the requested type when the ID list is
                    not an array.  If the 'out' parameter is specified data
                    are saved to the specified directory and an 'OK' string
                    is returned, or progress output in verbose=True.

        Example return values:
            [<class 'numpy.ndarray'>, <class 'numpy.ndarray'>, ....]
            [<class 'pandas.core.frame.DataFrame'>, ....]
                        :               :           :

        Examples:
            1) Retrieve spectra individually:

                id_list = spec.query (0.125, 12.123, 0.1)
                for id in id_list:
                    spec = spec.getSpec (id)
                    .... do something

            2) Retrieve spectra in bulk:

                spec = spec.getSpec (id_list, fmt='numpy')
                .... 'spec' is an array of NumPy objects that may be
                     different sizes

            3) Align spectra to a common wavelength grid, zero-padding on
               each side as needed:

                spec = spec.getSpec (id_list, fmt='numpy', align=True)
                .... 'spec' is an array of zero-padded NumPy objects

            4) Cutout the region 6500-7200A from a list of spectra:

                spec = spec.getSpec (id_list, cutout='6500-7200')
                .... 'spec' is an array of NumPy objects clipped to the
                     specified region and aligned as necessary



    Plot Interface

        The plot interface is used to retrieve preview graphics of the 
        spectra in the list [via plotGrid() or stackedImage()] or a single
        spectrum [via plot(), preview() or interactively with prospect()].

        When getting multiple spectra, a grid of preview plots of size (nx,ny)
        can be requested and will be constructed on the server and returned 
        as a single PNG file. If the 'page' parameter is set the caller can
        paginate through a list longer than the number of plot in the grid.
        If the 'align' parameter is set, spectra will be aligned to a common
        wavelength grid before plotting.

        The stackedImage() method can be used to return an image array
        in which each spectrum is a row in an output 2-D image.  The 'fmt'
        parameter can be used to request a PNG format, or a raw pixel array.
        The image is created in the same order as the ID list with the first
        ID being the bottom row of the image unless the 'yflip' parameter is
        enabled.  Input ID lists should be sorted (e.g. by redshift) before
        calling this method if some specific order is required.

        The DL user auth token is passed automatically in the X-DL-AuthToken
        header, it may be overridden with the 'token' kw parameter.

                        plot  (data, context=None, profile=None,
                               out=None, **kw)

                                Utility to batch plot a single spectrum, 
                                displays plot directly.  If 'out' specified,
                                plot saved as PNG file.

                                kw parameters:
                                    sky=False     Overplot sky lines
                                    model=False   Overplot model spectrum
                                    lines=<dict>  Mark spectral lines

           status = prospect  (data, context=None, profile=None, **kw)

                                Utility wrapper to launch the interactive
                                PROSPECT tool. [Not Yet Implemented]

                                kw parameters:
                                    TBD

             image = preview  (id, context=None, profile=None, **kw)

                                Return a single PNG preview plot of spectrum.

            image = plotGrid  (id_list, nx, ny, page=<N>,
                               context=None, profile=None, **kw)

                                Return an nx X ny grid of preview plots as
                                single PNG image.

        image = stackedImage  (id_list, fmt='png|numpy', 
                               align=False, yflip=False,
                               context=None, profile=None, **kw)

                                Return an image of all spectra in the list
                                rendered as rows in an image.

                where:
                    id          Single-object ID (context-specific)
                                Must be one of:
                                    - single string/int/int64/tuple identifier
                    id_list     List of object IDs (dataset-specific).
                                Must be one of:
                                    - single string/int/int64/tuple identifier
                                    - python array/list object
                                    - string of identifier values (one/line)
                                    - filename containing identifiers (one/line)
                                    - VOS name containing identifiers (one/line)
                    nx, ny      No. plots in each dim of grid
                    align       Align spectra to common wavelength grid
                    page        Get requested page in grid plot list
                    yflip       Flip stacked image in Y dimension
                    fmt         Result format
                    context     Dataset context
                    profile     Service profile
                    kw          optional parameters

                returns:
                    A PNG file or raw image array.

        Examples:
            1) Display a preview plot a given spectrum:

                from IPython.display import display, Image
                display(Image(spec.preview(id),
                        format='png', width=400, height=100, unconfined=True))

            2) Display a 5x5 grid of preview plots for a list:

                npages = np.round((len(id_list) / 25) + (25 / len(id_list))
                for pg in range(npages):
                    data = spec.getGridPlot(id_list, 5, 5, page=pg)
                    display(Image(data, format='png',
                            width=400, height=100, unconfined=True))

            3) Display a stacked image of spectra:

                from IPython.display import display, Image
                display(Image(spec.stackedImage(id_list, fmt='png'),
                        format='png', unconfined=True))

    Utility Methods

            df = to_pandas  (npy_data)      # convert to Pandas DataFram
    spec1d = to_Spectrum1D  (npy_data)      # convert to specutils Spectrum1D
            tab = to_Table  (npy_data)      # convert to Astropy Table



## Service Endpoints

The backend service provides the following endpoints:

    /                   GET     isAlive() or ping()
    /profiles           GET     Return service profiles
    /contexts           GET     Return dataset contexts
    /catalogs           GET     Return context catalogs

    /getSpec            POST    Get spectra for given ID list
    /preview            POST    Get preview plots for given ID list
    /gridPlot           POST    Get grid of preview plots for given ID list
    /stackedImage       POST    Get stacked image of given ID list
    /listSpan           POST    Get wavelength limits of array of IDs

    /validate           GET     Validate a context/profile value w/ server
    /available          GET     Service availability
    /shutdown           GET     Shutdown the service
    /debug              GET     Toggle debug flag

