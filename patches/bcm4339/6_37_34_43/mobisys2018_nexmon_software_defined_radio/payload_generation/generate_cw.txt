%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%  Nexmon SDR – Continuous‑Wave (CW) tone generator for 2.4055 GHz
%  Creates cw.sh that uploads an 80‑sample IQ buffer and starts *endless*
%  playback (NEX_SDR_START_TRANSMISSION with “loop”=1).
%
%  Author:  (adapted from SEEMOO Nexmon SDR example code)
%  ------------------------------------------------------------------------
%  Carrier maths:
%    • Wi‑Fi channel 1 LO = 2 412 MHz  (chanspec 0x1001)
%    • Baseband tone      = –6.5 MHz
%         2 412 MHz – 6.5 MHz  = 2 405.5 MHz
%    • fs (20 MHz channel) = 40 MS/s  (Broadcom 2× oversampling) :contentReference[oaicite:0]{index=0}
%    • 13 cycles of 6.5 MHz @ 40 MS/s = 13 · 6.153846 ≈ **80 samples** exactly.
%      Buffer therefore tiles perfectly → no discontinuity, no 6.7 kHz spurs
%      when the chip loops it :contentReference[oaicite:1]{index=1}
%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

clear; clc;

%% ----- user‑adjustable parameters ---------------------------------------
fs          = 40e6;          % sample‑rate inside BCM4339 (20 MHz Wi‑Fi BW)
tone_freq   = -6.5e6;        % baseband offset (‑ sign → below LO)
N           = 80;            % 13 full cycles fit exactly
amp_iq16    = 30000;         % |I|,|Q| ≤ 32767  (keeps PA linear)
power_index = 40;            % 0 = max power, larger = lower
start_offset_bytes = 0;      % store IQ buffer at Template RAM address 0
%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

%% 0. Helper for packing I/Q into 32‑bit words
pack_iq = @(i,q) bitor(bitshift(bitand(int32(i),hex2dec('FFFF')),16), ...
                        bitand(int32(q), hex2dec('FFFF')));

%% 1. Create seamless complex sinusoid (column vector)
t      = (0:N-1).' / fs;
cw     = exp(1j*2*pi*tone_freq .* t);

%% 2. Quantise to int16 and pack to int32
iq_i16 = int32(real(cw).*amp_iq16);
iq_q16 = int32(imag(cw).*amp_iq16);
iq32   = arrayfun(pack_iq, iq_i16, iq_q16);

%% 3. Build ioctl payloads -------------------------------------------------
NEX_WRITE_TEMPLATE_RAM     = 426;
NEX_SDR_START_TRANSMISSION = 427;

% ---- 3a.  write‑template‑RAM header + data (single block; <1500 B) -----
hdr      = [ int32(start_offset_bytes); ...
             int32(4*N); ...
             iq32(:) ];
b64_hdr  = matlab.net.base64encode(typecast(hdr,'uint8'));

% ---- 3b.  start‑transmission parameters ---------------------------------
sdr_par  = [ int32(N); ...                        % #samples
             int32(start_offset_bytes/4); ...     % offset in 32‑bit words
             int32(hex2dec('1001'));  ...         % chanspec: ch 1 / 20 MHz
             int32(power_index);     ...          % TX gain index
             int32(1) ];            % ------>  1 = *loop forever*  <------
b64_start = matlab.net.base64encode(typecast(sdr_par,'uint8'));

%% 4. Emit the phone‑side shell script ------------------------------------
fid = fopen('cw.sh','w');
fprintf(fid,'#!/system/bin/sh\n\n');
fprintf(fid,'# Upload IQ buffer (80 samples)\n');
fprintf(fid,'nexutil -s%d -b -l1500 -v%s\n', ...
            NEX_WRITE_TEMPLATE_RAM, b64_hdr);
fprintf(fid,'# Start endless playback\n');
fprintf(fid,'nexutil -s%d -b -l20 -v%s\n', ...
            NEX_SDR_START_TRANSMISSION, b64_start);
fprintf(fid,'echo ">> CW started at 2.4055 GHz (loop mode) <<"\n');
fprintf(fid,'echo "Run: nexutil -s428  to stop"\n');
fclose(fid);
system('chmod +x cw.sh');
disp('Done.  Push cw.sh to the phone and run it as root.');

