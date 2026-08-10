# HEIC, HEIF, and DNG Image Support on Ubuntu

This guide configures Ubuntu to open HEIC, HEIF, and Adobe DNG images directly without converting them first. It uses gThumb as the viewer because it supports both `libheif` images and camera RAW files through `libraw`.

---

## Technical Specifications

* **Target OS:** Ubuntu 26.04 LTS (Resolute), amd64
* **HEIC/HEIF Viewer:** gThumb
* **DNG/RAW Viewer:** gThumb with `libraw`
* **HEVC Decoder:** `libheif-plugin-libde265`
* **AV1 HEIF Decoder:** `libheif-plugin-aomdec`
* **File Manager Thumbnails:** `heif-thumbnailer`

---

## Format Overview

* **HEIC/HEIF:** A compressed image container commonly used by Apple and Android devices. Most HEIC photos use HEVC compression, while some HEIF images use AV1.
* **DNG:** Adobe Digital Negative, a camera RAW format. DNG files contain minimally processed sensor data and need a RAW-aware viewer.

These are image formats. A `.dng` file is unrelated to a macOS `.dmg` disk image.

---

## Step 1: Install Direct-Viewing Support

Install gThumb, HEIC integration, thumbnail support, and both common HEIF decoders:

```bash
sudo apt update
sudo apt install gthumb heif-gdk-pixbuf heif-thumbnailer \
  libheif-plugin-libde265 libheif-plugin-aomdec
```

The gThumb package installs `libraw`, which provides direct DNG and other camera RAW decoding.

---

## Step 2: Set gThumb as the Default Viewer

Configure gThumb only for DNG, HEIC, and HEIF. This does not change the default viewer for JPEG or PNG files.

```bash
xdg-mime default org.gnome.gThumb.desktop image/x-adobe-dng
xdg-mime default org.gnome.gThumb.desktop image/heic
xdg-mime default org.gnome.gThumb.desktop image/heif
```

Close and reopen Ubuntu Files after changing the associations. Double-clicking a supported image should now open it directly in gThumb.

---

## Step 3: Verify the Setup

Check that the required packages are installed:

```bash
dpkg-query -W -f='${binary:Package}: ${Status} ${Version}\n' \
  gthumb heif-gdk-pixbuf heif-thumbnailer \
  libheif-plugin-libde265 libheif-plugin-aomdec
```

Check the default applications:

```bash
printf 'DNG:  '; xdg-mime query default image/x-adobe-dng
printf 'HEIC: '; xdg-mime query default image/heic
printf 'HEIF: '; xdg-mime query default image/heif
```

All three commands should return:

```text
org.gnome.gThumb.desktop
```

Check the detected MIME type of a specific image:

```bash
gio info -a standard::content-type "/path/to/photo.heic"
gio info -a standard::content-type "/path/to/photo.dng"
```

---

## Optional: Advanced DNG Processing

gThumb is sufficient for browsing and directly viewing most DNG files. Install RawTherapee if you need exposure correction, lens profiles, denoising, color processing, or support for an unusual camera RAW variant:

```bash
sudo apt install rawtherapee
```

RawTherapee is a full RAW processor rather than a lightweight everyday image viewer.

---

## Troubleshooting

### HEIC Opens as an Unsupported Format

Confirm that the HEVC decoder is installed:

```bash
dpkg-query -W libheif-plugin-libde265
```

Most phone HEIC files require this plugin. AV1-based HEIF images require `libheif-plugin-aomdec`.

### DNG Opens as an Unsupported Format

Try opening it explicitly with gThumb:

```bash
gthumb "/path/to/photo.dng"
```

If that fails, install RawTherapee and test the file there. Some recent phones and cameras produce DNG variants that require a newer or more specialized RAW decoder.

### The Wrong Application Opens

Reapply the MIME association and verify it:

```bash
xdg-mime default org.gnome.gThumb.desktop image/x-adobe-dng
xdg-mime default org.gnome.gThumb.desktop image/heic
xdg-mime default org.gnome.gThumb.desktop image/heif
```

You can also right-click the image in Files, select **Open With**, choose **gThumb**, and make it the default.

### Thumbnails Do Not Appear in Files

Clear only the thumbnail cache and restart Files:

```bash
rm -rf ~/.cache/thumbnails/*
nautilus -q
```

Open Files again and allow a few moments for thumbnails to regenerate.

---

## Remove the Additional Support

To remove gThumb and the extra HEVC decoder:

```bash
sudo apt remove gthumb libheif-plugin-libde265
sudo apt autoremove
```

Ubuntu's original image viewer remains installed.