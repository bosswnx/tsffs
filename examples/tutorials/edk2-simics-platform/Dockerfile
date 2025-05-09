FROM ghcr.io/tianocore/containers/fedora-37-build:a0dd931@sha256:09843c8043412b49c36bd3daaa0660210b3bcd60a8417fe9f9e24f986cd345ff

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG PROJECT=

WORKDIR "$PROJECT"

RUN git clone https://github.com/tianocore/edk2.git && \
    git clone https://github.com/tianocore/edk2-platforms.git && \
    git clone https://github.com/tianocore/edk2-non-osi.git && \
    git clone https://github.com/IntelFsp/FSP.git

ARG EDK2_HASH="eccdab6"
ARG EDK2_PLATFORMS_HASH="f446fff"
ARG EDK2_NON_OSI_HASH="1f4d784"
ARG INTEL_FSP_HASH="8beacd5"

RUN git -C edk2 checkout "${EDK2_HASH}" && \
    git -C edk2 submodule update --init && \
    git -C edk2-platforms checkout "${EDK2_PLATFORMS_HASH}" && \
    git -C edk2-platforms submodule update --init && \
    git -C edk2-non-osi checkout "${EDK2_NON_OSI_HASH}" && \
    git -C edk2-non-osi submodule update --init && \
    git -C FSP checkout "${INTEL_FSP_HASH}" && \
    git -C FSP submodule update --init

ENV EDK_TOOLS_PATH="${PROJECT}/edk2/BaseTools/"
ENV PACKAGES_PATH="${PROJECT}/edk2:${PROJECT}/edk2-platforms:${PROJECT}/edk2-non-osi"
ENV WORKSPACE="${PROJECT}"

RUN source edk2/edksetup.sh && \
    make -C edk2/BaseTools/

WORKDIR "${PROJECT}/edk2-platforms/Platform/Intel"

COPY <<'EOF' /tmp/edk2-platforms.patch
diff --git a/Platform/Intel/SimicsOpenBoardPkg/Library/DxeLogoLib/Logo.c b/Platform/Intel/SimicsOpenBoardPkg/Library/DxeLogoLib/Logo.c
index 9cea5f4665..00815adba2 100644
--- a/Platform/Intel/SimicsOpenBoardPkg/Library/DxeLogoLib/Logo.c
+++ b/Platform/Intel/SimicsOpenBoardPkg/Library/DxeLogoLib/Logo.c
@@ -11,6 +11,7 @@
 #include <OemBadging.h>
 #include <Protocol/GraphicsOutput.h>
 #include <Library/BaseLib.h>
+#include <Library/DebugLib.h>
 #include <Library/UefiLib.h>
 #include <Library/BaseMemoryLib.h>
 #include <Library/UefiBootServicesTableLib.h>
@@ -22,6 +23,7 @@
 #include <IndustryStandard/Bmp.h>
 #include <Protocol/BootLogo.h>
 
+#include "tsffs.h"
 /**
   Convert a *.BMP graphics image to a GOP blt buffer. If a NULL Blt buffer
   is passed in a GopBlt buffer will be allocated by this routine. If a GopBlt
@@ -164,9 +166,6 @@ ConvertBmpToGopBlt (
     *GopBltSize = (UINTN) BltBufferSize;
     *GopBlt     = AllocatePool (*GopBltSize);
     IsAllocated = TRUE;
-    if (*GopBlt == NULL) {
-      return EFI_OUT_OF_RESOURCES;
-    }
   } else {
     //
     // GopBlt has been allocated by caller.
@@ -184,6 +183,7 @@ ConvertBmpToGopBlt (
   // Convert image from BMP to Blt buffer format
   //
   BltBuffer = *GopBlt;
+  ASSERT (BmpHeader->PixelHeight != 0);
   for (Height = 0; Height < BmpHeader->PixelHeight; Height++) {
     Blt = &BltBuffer[(BmpHeader->PixelHeight - Height - 1) * BmpHeader->PixelWidth];
     for (Width = 0; Width < BmpHeader->PixelWidth; Width++, Image++, Blt++) {
@@ -398,6 +398,7 @@ EnableBootLogo (
     // Try BMP decoder
     //
     Blt = NULL;
+    HARNESS_START(ImageData, &ImageSize);
     Status = ConvertBmpToGopBlt (
               ImageData,
               ImageSize,
@@ -411,6 +412,7 @@ EnableBootLogo (
       FreePool (ImageData);
 
       if (Badging == NULL) {
+        HARNESS_STOP();
         return Status;
       } else {
         continue;
@@ -537,6 +539,7 @@ Done:
       FreePool (Blt);
     }
 
+    HARNESS_STOP();
     return Status;
   }
 
@@ -561,6 +564,7 @@ Done:
     // Ensure the LogoHeight * LogoWidth doesn't overflow
     //
     if (LogoHeight > DivU64x64Remainder ((UINTN) ~0, LogoWidth, NULL)) {
+      HARNESS_STOP();
       return EFI_UNSUPPORTED;
     }
     BufferSize = MultU64x64 (LogoWidth, LogoHeight);
@@ -569,11 +573,13 @@ Done:
     // Ensure the BufferSize * sizeof (EFI_GRAPHICS_OUTPUT_BLT_PIXEL) doesn't overflow
     //
     if (BufferSize > DivU64x32 ((UINTN) ~0, sizeof (EFI_GRAPHICS_OUTPUT_BLT_PIXEL))) {
+      HARNESS_STOP();
       return EFI_UNSUPPORTED;
     }
 
     LogoBlt = AllocateZeroPool ((UINTN)BufferSize * sizeof (EFI_GRAPHICS_OUTPUT_BLT_PIXEL));
     if (LogoBlt == NULL) {
+      HARNESS_STOP();
       return EFI_OUT_OF_RESOURCES;
     }
 
@@ -600,5 +606,6 @@ Done:
   }
   FreePool (LogoBlt);
 
+  HARNESS_STOP();
   return Status;
 }

EOF

COPY tsffs.h "${PROJECT}/edk2-platforms/Platform/Intel/SimicsOpenBoardPkg/Library/DxeLogoLib/tsffs.h"

RUN git -C "${PROJECT}/edk2-platforms" apply --whitespace fix /tmp/edk2-platforms.patch

# Build SimicsOpenBoardPkg
RUN source "${PROJECT}/edk2/edksetup.sh" && \
    python build_bios.py -p BoardX58Ich10X64 -d -t GCC
    
