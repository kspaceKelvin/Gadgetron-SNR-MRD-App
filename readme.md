#  SNR Scaled Reconstruction with Gadgetron
The [Gadgetron](http://gadgetron.github.io/) implements a Signal-to-Noise Ratio (SNR) scaled reconstruction as described by Kellman P *et al* in [Image reconstruction in SNR units: A general method for SNR measurement](https://onlinelibrary.wiley.com/doi/10.1002/mrm.20713).

## Input Data
This workflow takes as raw k-space data as input and outputs GRAPPA reconstructed images, corresponding g-factor maps, and SNR scaled images.  Noise data (generally in the form of dependent data) is required for correct SNR scaling.

## Supported Configurations
The [default_measurement_dependencies.xml](https://github.com/gadgetron/gadgetron/blob/master/gadgets/mri_core/config/default_measurement_dependencies.xml) config is used to process the noise dependencies necessary for SNR scaling.  The SNR-scaled reconstruction itself in Gadgetron uses the [Generic_Cartesian_Grappa_SNR.xml](https://github.com/gadgetron/gadgetron/blob/master/gadgets/mri_core/generic_recon_gadgets/config/Generic_Cartesian_Grappa_SNR.xml) config.  

## Running the app
A Gadgetron image can be downloaded from Docker Hub at https://hub.docker.com/u/gadgetron.  In a command prompt on a system with [Docker](https://www.docker.com/) installed, download the Docker image:
```
docker pull ghcr.io/gadgetron/gadgetron/gadgetron_ubuntu_rt_nocuda
```

Start the Docker image and share port 9002:
```
docker run --rm -p 9002:9002 ghcr.io/gadgetron/gadgetron/gadgetron_ubuntu_rt_nocuda
```

In another window, use an MRD client such as the one provided from the [python-ismrmrd-server](https://github.com/kspaceKelvin/python-ismrmrd-server#11-reconstruct-a-phantom-raw-data-set-using-the-mrd-clientserver-pair) or the [gadgetron_ismrmrd_client](https://gadgetron.readthedocs.io/en/latest/using.html#server-mode):

Run the client and send the noise data to the server for processing:
```
gadgetron_ismrmrd_client -c default_measurement_dependencies.xml -g dataset_1 -f rawdata.mrd
```

Run the client and send the imaging k-space data to the server for SNR scaled reconstruction:
```
gadgetron_ismrmrd_client -c Generic_Cartesian_Grappa_SNR.xml -g dataset_2 -o recon.mrd -f rawdata.mrd
```

The output file (e.g. recon.mrd) contains 3 groups of images:
- ``image_1`` contains GRAPPA reconstructed images
- ``image_20`` contains the g-factor maps, scaled up by a factor of 100
- ``image_300`` contains the SNR scaled images, scaled up by a factor of 10

## Converting Data From Siemens Raw Data
Siemens "twix" raw data can be converted into MRD format using the [siemens_to_ismrmrd](https://github.com/ismrmrd/siemens_to_ismrmrd/) converter.  Run the converter with the following options:
``
siemens_to_ismrmrd -Z -M --skipSyncData -f twix.dat
``
where:
- ``-Z`` converts all (i.e. including dependent data) datasets from a multi-raid twix .dat file.
- ``-M`` stores these multiple datasets into a single MRD file with multiple group named ``dataset_1``, ``dataset_2``, etc.  The last dataset is the primary measurement data and the earlier datasets are dependent data (e.g. containing noise measurements)
- ``--skipSyncData`` is required for datasets from software version XA31 and above.

Note that the Gadgetron requires that the [patientID](https://github.com/ismrmrd/ismrmrd/blob/d805117b0d2075c8b6c4473eac55b055d2ba9590/schema/ismrmrd.xsd#L26) field be present and not empty for noise dependency data to be properly stored.  The following Python code can be used to populate the ``patientID`` with a dummy value if needed:

```python
import ismrmrd
dset = ismrmrd.Dataset('dataset.mrd', 'dataset_1', False)

xml_header = dset.read_xml_header()
metadata = ismrmrd.xsd.CreateFromDocument(xml_header.decode("utf-8"))

if metadata.subjectInformation is None:
    metadata.subjectInformation = ismrmrd.xsd.subjectInformationType()

metadata.subjectInformation.patientID = 'dummy'

dset.write_xml_header(bytes(metadata.toXML(), 'utf-8'))
dset.close()
```