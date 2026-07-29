# SPDX-License-Identifier: GPL-2.0

CROSS_COMPILE ?= aarch64-linux-gnu-
CC ?= $(CROSS_COMPILE)gcc
CFLAGS = -Wall -Wextra -O2 -g -static

all: mdm_helper_native pm_service_native sysmon_probe_native dms_nas_probe diag_reader ipa_qmi_proxy xiaomi_efs_probe diag_efs_write efs_mhi_write mhi_efs_sync qrtr_tftp_server

mdm_helper_native: mdm_helper_native.c
	$(CC) $(CFLAGS) -o $@ $<

pm_service_native: pm_service_native.c
	$(CC) $(CFLAGS) -o $@ $<

sysmon_probe_native: sysmon_probe_native.c
	$(CC) $(CFLAGS) -o $@ $<

dms_nas_probe: dms_nas_probe.c
	$(CC) $(CFLAGS) -o $@ $<

diag_reader: diag_reader.c
	$(CC) $(CFLAGS) -o $@ $<

ipa_qmi_proxy: ipa_qmi_proxy.c
	$(CC) $(CFLAGS) -o $@ $<

xiaomi_efs_probe: xiaomi_efs_probe.c
	$(CC) $(CFLAGS) -o $@ $<

diag_efs_write: diag_efs_write.c
	$(CC) $(CFLAGS) -o $@ $<

efs_mhi_write: efs_mhi_write.c
	$(CC) $(CFLAGS) -o $@ $<

clean:
	$(RM) mdm_helper_native pm_service_native sysmon_probe_native dms_nas_probe diag_reader ipa_qmi_proxy xiaomi_efs_probe diag_efs_write efs_mhi_write

.PHONY: all clean

mhi_efs_sync: mhi_efs_sync.c
	$(CC) $(CFLAGS) -o $@ $<

qrtr_tftp_server: qrtr_tftp_server.c
	$(CC) $(CFLAGS) -o $@ $<
