Coregistration
==============

A common raster operation, particularly when developing training datasets for machine learning models, is to take a number of input rasters and then warp them to the projection, grid, and bounds of a reference raster that they all intersect.

This topic builds on a few others -- check them out before continuing:

   * :doc:`Reprojection <reproject>`
   * :doc:`Resampling <resampling>`
   * :doc:`Virtual Warping <virtual-warping>`

A basic implementation of this operation requires the following steps:

   * Open the reference raster as a :class:`rasterio.vrt.WarpedVRT()`
   * Read the projection (CRS), dimensions, and extent from the reference raster
   * Open the input rasters as :class:`rasterio.vrt.WarpedVRT()`s using the retrieved values from the reference raster
   * Read data from each input raster
   * Perform a pixel-wise merge of data for overlapping pixels in each input raster
   * Write the results to a new raster

In addition to the above steps, users may wish to customize the resample and merge operations during the coregister operation. Rasterio supplies a number of built-in resampling methods in :class:`rasterio.enums.Resampling()`. Rasterio's :func:`rasterio.merge.merge()` provides a number of built-in operations or allows users to provide their own pixel-wise merge callable.

Putting it all together:

.. code-block:: python

    def coregister(from_uris, to_uri, dest_file, resampling=Resampling.bilinear, merge_method="first"):
        with rasterio.open(to_uri) as ds_to:
            from_ds_list = [rasterio.open(uri) for uri in from_uris]
            vrts = [
                WarpedVRT(
                    ds,
                    crs=ds_to.crs,
                    height=ds_to.height,
                    width=ds_to.width,
                    resampling=resampling,
                    transform=ds_to.transform,
                )
                for ds in from_ds_list
            ]
            out_nodata = ds_to.nodata
            data, out_transform = rasterio_merge(vrts, method=merge_method)
            for vrt in vrts:
                vrt.close()
            for ds in from_ds_list:
                ds.close()
            with rasterio.open(
                dest_file,
                "w",
                compress="lzw",
                count=1,
                crs=ds_to.crs,
                driver="GTiff",
                dtype=data.dtype,
                height=data.shape[1],
                width=data.shape[2],
                nodata=out_nodata,
                tiled=True,
                transform=out_transform,
            ) as dst:
                dst.write(data[0], indexes=1)

With this method, the user can choose their resample and merge operations, and WarpedVRTs can be read from any backend supported by Rasterio, such as AWS S3. Be careful when loading very large reference rasters, as the merge result is loaded in memory before being saved. More advanced implementations could improve memory usage, read multiple bands, and support alternate output formats.
