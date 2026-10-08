# AIKON server: Combined builder stage for ffmpeg and Go to stay under 2 stages limit
FROM golang:1.26-alpine@sha256:8ac98ca534ac3f51e1f420a1dd2c15e74c75cfa0f23f3ad27eb5d7236c349a0c AS builder
RUN apk add --no-cache build-base openssl

# Build ffmpeg
ADD --checksum=sha256:8c3850283eb25fa026482078a04051e0be17347b09ef81a0849bec15a96e002e \
    https://ffmpeg.org/releases/ffmpeg-9.0.2.tar.xz /src/ffmpeg.tar.xz
WORKDIR /src
RUN tar xf ffmpeg.tar.xz && cd ffmpeg-9.0.2 \
 && ./configure --disable-everything --disable-autodetect --disable-network --disable-doc --disable-debug \
      --disable-ffplay --disable-ffprobe --disable-x86asm --enable-static --disable-shared \
      --extra-ldflags=-static --enable-swresample \
      --enable-demuxer=amr,wav --enable-parser=amr --enable-protocol=pipe \
      --enable-decoder=amrnb,pcm_s16le,pcm_u8,pcm_alaw,pcm_mulaw,adpcm_ima_wav \
      --enable-encoder=pcm_s16le --enable-muxer=pcm_s16le \
      --enable-filter=aresample,aformat,anull,atrim,abuffer,abuffersink >/dev/null \
 && make -j"$(nproc)" ffmpeg >/dev/null && strip ffmpeg && cp ffmpeg /ffmpeg

# Ensure ffmpeg is available for Go tests
RUN cp /ffmpeg /usr/local/bin/ffmpeg

# Build Go server
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY *.go ./
COPY lang ./lang
RUN go vet ./... && go test -count=1 ./... \
 && CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -buildid=" -o /out/claude-s40-server . \
 && mkdir -p /out/data

# Final Stage
FROM gcr.io/distroless/static-debian12:nonroot@sha256:afa5c872c891853ca7fcf1f12c3edb23f7eeef36189728842dd51042ff57f7ab
COPY --from=builder /out/claude-s40-server /claude-s40-server
COPY --from=builder /ffmpeg /usr/local/bin/ffmpeg
# empty, nonroot-owned /data so a fresh named volume is writable
COPY --from=builder --chown=65532:65532 /out/data /data
USER nonroot:nonroot
EXPOSE 8443 9090
ENTRYPOINT ["/claude-s40-server"]
