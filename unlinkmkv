#!/usr/bin/env perl
# UnlinkMKV - Undo segment linking in MKV files
# Garret Noling <garret@werockjustbecause.com> 2013

require 5.010;
use strict;
use XML::LibXML;
use File::Glob     qw/:globally :nocase/;
use Math::BigFloat qw/:constant/;
use Getopt::Long   qw/:config passthrough/;
use Log::Log4perl  qw/:easy/;
use File::Basename;
use String::CRC32;
use IPC::Open3;
use IO::Select;
use Symbol;
use Cwd            qw/cwd realpath abs_path/;

    my $loglevel = 'INFO';
    GetOptions (
	'll|loglevel=s' => \$loglevel,
    );
    my $conf = qq(
	log4perl.logger                   = $loglevel, STDINF
	log4perl.appender.STDINF          = Log::Log4perl::Appender::ScreenColoredLevels
	log4perl.appender.STDINF.stderr   = 0
	log4perl.appender.STDINF.layout   = PatternLayout
	log4perl.appender.STDINF.layout.ConversionPattern = %x%m{chomp}%n
    );
    Log::Log4perl->init_once(\$conf);
    Log::Log4perl::NDC->push("");
    INFO "UnlinkMKV";
    UnlinkMKV::more();

    my $out_dir        = '/home/garret/Desktop/UnlinkMKV';
    my $tmpdir         = "/home/garret/tmp/umkv";
    my $ffmpeg         = "/opt/ffmpeg/bin/ffmpeg";
    my $mkvext         = "/usr/bin/mkvextract";
    my $mkvinfo        = "/usr/bin/mkvinfo";
    my $mkvmerge       = "/usr/bin/mkvmerge";
    my $fix_audio      = 0;
    my $fix_video      = 0;
    my $fix_subtitles  = 1;
    my $ignoredef      = 0;
    my $chapters       = 1;
    my $playresx;
    my $playresy;

    GetOptions (
	'tmpdir=s'            => \$tmpdir,
	'fix-audio|fa!'       => \$fix_audio,
	'fix-video|fv!'       => \$fix_video,
	'fix-subtitles|fs!'   => \$fix_subtitles,
	'outdir=s'            => \$out_dir,
	'playresx=i'          => \$playresx,
	'playresy=i'          => \$playresy,
	'ignore-default-flag' => \$ignoredef,
	'chapters!'           => \$chapters,
    );

    $out_dir = abs_path($out_dir);

    my $UMKV = UnlinkMKV->new({
	ffmpeg            => $ffmpeg,
	mkvext            => $mkvext,
	mkvinfo           => $mkvinfo,
	mkvmerge          => $mkvmerge,
	tmp               => $tmpdir,
	fixaudio          => $fix_audio,
	fixvideo          => $fix_video,
	fixsubs           => $fix_subtitles,
	outdir            => $out_dir,
	playresx          => $playresx,
	playresy          => $playresy,
	ignoredefaultflag => $ignoredef,
	chapters          => $chapters,
    });

    my @LIST;
    foreach my $item (@ARGV) {
	if(-d $item) {
	    opendir my $D, $item;
	    while (my $F = readdir($D)) {
		if (-f "$item/$F" && $F =~ /\.mkv$/i && !-f "$out_dir/$F") {
		    push @LIST, abs_path("$item/$F");
		}
	    }
	    closedir $D;
	}
	elsif(-f $item) {
	    push @LIST, abs_path($item);
	}
    }
    do { $UMKV->process($_) } for @LIST;
    exit;

package UnlinkMKV {
use strict;
use XML::LibXML;
use File::Glob     qw/:globally :nocase/;
use Math::BigFloat qw/:constant/;
use Getopt::Long   qw/:config passthrough/;
use Log::Log4perl  qw/:easy/;
use File::Basename;
use String::CRC32;
use IPC::Open3;
use IO::Select;
use Symbol;
use Cwd            qw/cwd realpath abs_path/;

    sub new {
        my $type      = shift;
	my $opt       = shift;
        my ($self)    = {};
        bless($self, $type);
	$self->{opt}  = $opt;
	$self->{xml}  = XML::LibXML->new();
	return $self;
    }

    sub DESTROY {
	my $self = shift;
	$self->sys("/bin/rm",    "-fr", $self->{tmp});
	DEBUG "removed tmp $self->{tmp} [exiting]";
    }

    sub mktmp {
	my $self = shift;
	if(!defined $self->{opt}->{tmp}) {
	    $self->{tmp} = "/tmp/unlinkmkv/$$";
	}
	else {
	    $self->{tmp} = $self->{opt}->{tmp};
	    $self->{tmp} =~ s/\/$//;
	    $self->{tmp} .= "/$$";
	}
	$self->sys("/bin/rm",    "-fr", $self->{tmp});
	DEBUG "removed tmp $self->{tmp}";
	$self->sys('/bin/mkdir', '-p', "$self->{tmp}/attach");
	$self->sys('/bin/mkdir', '-p', "$self->{tmp}/parts");
	$self->sys('/bin/mkdir', '-p', "$self->{tmp}/encodes");
	$self->sys('/bin/mkdir', '-p', "$self->{tmp}/subtitles");
	$self->sys('/bin/mkdir', '-p', "$self->{tmp}/segments");
	DEBUG "created tmp $self->{tmp}";
    }

    sub process {
	my $self     = shift;
	my $item     = shift;
	my $origpath = dirname(abs_path($item));
	chdir($origpath);
	INFO "processing $item";
	more();
	INFO "checking if file is segmented";
	if($self->is_linked($item)) {
	    INFO "generating chapter file";
	    more();
	    my(@segments, @splits);
	    my($parent, $dir, $suffix) = fileparse($item, qr/\.[mM][kK][vV]/);
	    INFO "loading chapters";
	    more();
	    open my $H, '-|', $self->{opt}->{mkvext}, 'chapters', $item;
	    binmode $H;
	    my $xml = $self->{xml}->load_xml(IO => $H);
	    close $H;
	    open my $out_chapters_orig, '>', $self->{tmp} . "/$parent-chapters-original.xml";
	    print {$out_chapters_orig} $xml->toString;
	    close $out_chapters_orig;
	    my $offs_time_end            = '00:00:00.000000000';
	    my $last_time_end            = '00:00:00.000000000';
	    my $offset                   = '00:00:00.000000000';
	    my $chaptercount             = 1;
	    foreach my $edition ($xml->findnodes('//EditionFlagDefault[.=0]')) {
		if(!$self->{opt}->{ignoredefaultflag}) {
		    $edition->parentNode->unbindNode;
		    WARN "non-default chapter dropped";
		}
		else {
		    INFO "non-default chapter kept on purpose";
		}
	    }
	    foreach my $chapter ($xml->findnodes('//ChapterAtom')) {
		my ($ChapterTimeStart) = $chapter->findnodes('./ChapterTimeStart/text()');
		my ($ChapterTimeEnd)   = $chapter->findnodes('./ChapterTimeEnd/text()');
		if($chapter->exists('ChapterSegmentUID') && $chapter->findvalue('ChapterFlagEnabled') == 1) {
		    my ($SegmentUID, $SegmentELE, $SegmentUIDText);
		    ($SegmentELE)   = $chapter->findnodes('./ChapterSegmentUID');
		    ($SegmentUID)   = $chapter->findnodes('./ChapterSegmentUID/text()');
		    $SegmentUIDText = $SegmentUID->textContent();
		    if($SegmentELE->getAttribute('format') eq 'hex') {
			$SegmentUIDText =~ s/\n//g;
			$SegmentUIDText =~ s/\s//g;
			$SegmentUIDText =~ s/([a-zA-Z0-9]{2})/ 0x$1/g;
			$SegmentUIDText =~ s/^\s//;
		    }
		    elsif($SegmentELE->getAttribute('format') eq 'ascii') {
			$SegmentUIDText =~ s/(.)/sprintf("0x%x ",ord($1))/eg;
			$SegmentUIDText =~ s/\s$//;
		    }
		    push @segments, {
			start       => $ChapterTimeStart->textContent(),
			stop        => $ChapterTimeEnd->textContent(),
			id          => $SegmentUIDText,
			split_start => $last_time_end
		    };
		    push @splits, $last_time_end unless $last_time_end eq '00:00:00.000000000';
		    $offset = $self->add_duration_to_timecode($offset, $ChapterTimeEnd->textContent());
		    if($offs_time_end eq '00:00:00.000000000' && $chaptercount > 1) {
			$ChapterTimeStart->setData($offset);
			$ChapterTimeEnd->setData($self->add_duration_to_timecode($offset, $ChapterTimeEnd->textContent()));
		    }
		    else {
			$ChapterTimeStart->setData($offs_time_end);
			$ChapterTimeEnd->setData($self->add_duration_to_timecode($offs_time_end, $ChapterTimeEnd->textContent()));
		    }
		    $chapter->removeChild($chapter->findnodes('./ChapterSegmentUID'));
		    WARN sprintf("A %s, %s, %s, %s", $ChapterTimeStart->textContent(), $ChapterTimeEnd->textContent(), $offset, $offs_time_end);
		}
		else {
		    eval { if(defined $ChapterTimeEnd->textContent() && defined $ChapterTimeStart->textContent()){}; 1 } or next;
		    push @segments, {
			file        => $self->setpart(basename($item), abs_path($item)),
			start       => $ChapterTimeStart->textContent(),
			stop        => $ChapterTimeEnd->textContent(),
			split_start => $ChapterTimeStart->textContent(),
			split_stop  => $ChapterTimeEnd->textContent()
		    };
		    $last_time_end = $ChapterTimeEnd->textContent();
		    $ChapterTimeStart->setData($self->add_duration_to_timecode($ChapterTimeStart->textContent(), $offset));
		    $ChapterTimeEnd->setData($self->add_duration_to_timecode($ChapterTimeEnd->textContent(),     $offset));
		    $offs_time_end = $ChapterTimeEnd->textContent();
		    WARN sprintf("B %s, %s, %s, %s", $ChapterTimeStart->textContent(), $ChapterTimeEnd->textContent(), $offset, $offs_time_end);
		}
		$chaptercount++;
	    }
	    less();

	    INFO "writing chapter temporary file";
	    open my $out_chapters, '>', $self->{tmp} . "/$parent-chapters.xml";
	    print {$out_chapters} $xml->toString;
	    close $out_chapters;

	    INFO "looking for segment parts";
	    more();
	    foreach my $mkv (<*.mkv>) {
		$mkv = abs_path("$dir/$mkv");
		next if $mkv eq $item;
		my ($id, $dur) = $self->mkvinfo($mkv);
		for (@segments) {
		    next unless defined $_->{id};
		    if($_->{id} eq $id && basename($mkv) ne basename($item)) {
			$_->{file} = $self->setpart(basename($mkv), $mkv);
			INFO "found part $_->{file}";
		    }
		}
	    }
	    less();

	    INFO "checking that all required segments were found";
	    my $okay_to_proceed = 1;
	    for (@segments) {
		if(defined $_->{id} && !defined $_->{file}) {
		    DEBUG "missing segment: $_->{id}";
		    $okay_to_proceed = 0;
		}
	    }
	    if(!$okay_to_proceed) {
		WARN "missing segments!";
		return;
	    }

	    INFO "extracting attachments";
	    more();
	    foreach my $seg (@segments) {
		my $file = "$seg->{file}";
		my $in = 0;
		my ($N, $T, $D, $U);
		TRACE "FILE $file";
		for(split /\n/, $self->sys($self->{opt}->{mkvinfo}, $file)) {
		    chomp;
		    if ($_ =~ /^\| \+ Attached/) {
			$in = 1;
		    }
		    elsif($in && $_ =~ /File name: (.*)/) {
			$N = $1;
		    }
		    elsif($in && $_ =~ /Mime type: (.*)/) {
			$T = $1;
		    }
		    elsif($in && $_ =~ /File data, size: (.*)/) {
			$D = $1;
		    }
		    elsif($in && $_ =~ /File UID: (.*)/) {
			$U = $1;
		    }
		    if (defined $N && defined $T && defined $D && defined $U) {
			push @{$seg->{attachments}}, { name => $N, type => $T, data => $D, UID => $U };
			undef $N;
			undef $T;
			undef $D;
			undef $U;
		    }
		}
		close $H;
		if (defined @{$seg->{attachments}} && @{$seg->{attachments}} > 0) {
		    my $dir = `pwd`;
		    TRACE "chdir $self->{tmp}/attach";
		    chdir("$self->{tmp}/attach");
		    $self->sys($self->{opt}->{mkvext}, 'attachments', $file, (1..$#{$seg->{attachments}}+1));
		    chdir("$dir");
		}
	    }
	    less();

	    my @atts;
	    foreach my $att (split /\n/, `find $self->{tmp}/attach -type f`) {
		push @atts, ('--attachment-mime-type', 'application/x-truetype-font', '--attach-file', $att);
	    }

	    INFO "creating splits";
	    more();
	    if(scalar(@splits) > 0) {
		DEBUG "splitting file: $item";
		$self->sys($self->{opt}->{mkvmerge}, '--no-chapters', '-o', "$self->{tmp}/parts/split-%03d.mkv", $item, '--split', 'timecodes:' . join(',',@splits));
	    }
	    less();

	    INFO "setting parts";
	    more();
	    my (@parts, $LAST);
	    my $count = 1;
	    foreach my $segment (@segments) {
		if(defined $segment->{id} && $segment->{start} =~ /^00:00:00\./ || ($LAST ne $segment->{file} && scalar(@splits) == 0)) {
		    DEBUG "part $segment->{file}";
		    push @parts, $segment->{file};
		}
		elsif($LAST ne $segment->{file}) {
		    my $f = sprintf("$self->{tmp}/parts/split-%03d.mkv",$count);
		    DEBUG "part $f";
		    push @parts, $f;
		    $count++;
		}
		$LAST = $segment->{file};
	    }
	    less();

	    my $subs;
	    if($self->{opt}->{fixsubs}) {
		INFO "extracting subs";
		more();
		foreach my $part (@parts) {
		    DEBUG "$part";
		    my $in  = 0;
		    my $sub = 0;
		    my ($N, $T, $D, $U);
		    for (split /\n/, $self->sys($self->{opt}->{mkvinfo}, $part)) {
			chomp;
			if ($_ =~ /^\| \+ A track/) {
			    $in = 1;
			    undef $N;
			    undef $T;
			    undef $D;
			    undef $U;
			    $sub = 0;
			}
			elsif($in && $_ =~ /Track type: subtitles/) {
			    $sub = 1;
			}
			elsif($in && $_ =~ /Track number: .*: (\d)\)$/) {
			    $T = $1;
			}
			if (defined $in && $sub && $T) {
			    $self->sys($self->{opt}->{mkvext}, 'tracks', $part, "$T:$self->{tmp}/subtitles/".basename($part)."-$T.ass");
			    push @{$subs->{$part}}, "$self->{tmp}/subtitles/".basename($part)."-$T.ass";
			    undef $T;
			    $in  = 0;
			    $sub = 0;
			}
		    }
		}
		less();

		INFO "making substyles unique";
		more();
		my $styles;
		foreach my $f (keys %$subs) {
		    push @$styles, @{$self->uniquify_substyles($subs->{$f})};
		}
		less();

		INFO "mashing unique substyles to all parts";
		more();
		foreach my $f (keys %$subs) {
		    $self->mush_substyles($subs->{$f}, $styles);
		}
		less();

		INFO "remuxing subtitles";
		more();
		foreach my $f (keys %$subs) {
		    DEBUG $f;
		    my @stracks;
		    foreach my $T (@{$subs->{$f}}) {
			push @stracks, $T;
		    }
		    $self->sys($self->{opt}->{mkvmerge}, '-o', "$f-fixsubs.mkv", '--no-chapters', '--no-subs', $f, @stracks, @atts);
		    $self->replace($f, "$f-fixsubs.mkv");
		}
		less();
	    }

	    if($self->{opt}->{fixvideo} || $self->{opt}->{fixaudio}) {
		INFO "encoding parts";
		more();
		foreach my $part (@parts) {
		    my @vopt = qw/-vcodec copy/;
		    my @aopt = qw/-map 0 -acodec copy/;
		    WARN $part;
		    if ($self->{opt}->{fixvideo}) {
			my $br = $self->bitrate($part);
			@vopt  = undef;
			@vopt  = ('-c:v', 'libx264', '-b:v', $br.'k', '-minrate', $br.'k', '-maxrate', ($br*2).'k', '-bufsize', '1835k');
		    }
		    if ($self->{opt}->{fixaudio}) {
			@aopt = undef;
			@aopt = qw/-map 0 -acodec ac3 -ab 320k/;
		    }
		    $self->sys($self->{opt}->{ffmpeg}, '-i', $part, @vopt, @aopt, "$part-fixed.mkv");
		    $self->replace($part, "$part-fixed.mkv");
		}
		less();
	    }

	    INFO "building file";
	    more();
	    my @PRTS;
	    foreach my $part (@parts) {
		push @PRTS, $part;
		push @PRTS, '+';
	    }
	    pop @PRTS;
	    if($self->{opt}->{chapters}) {
		$self->sys($self->{opt}->{mkvmerge}, '--no-chapters', '-M', '--chapters', $self->{tmp} . "/$parent-chapters.xml", '-o', "$self->{tmp}/encodes/".basename($item), @PRTS);
	    }
	    else {
		$self->sys($self->{opt}->{mkvmerge}, '--no-chapters', '-M', '--chapters', $self->{tmp} . "/$parent-chapters.xml", '-o', "$self->{tmp}/encodes/".basename($item), @PRTS);
	    }
	    less();

	    INFO "fixing subs, again... (maybe an mkvmerge issue?)";
	    more();
	    if($self->{opt}->{fixsubs}) {
		my @FS;
		my $in  = 0;
		my $sub = 0;
		my ($N, $T, $D, $U);
		for (split /\n/, $self->sys($self->{opt}->{mkvinfo}, "$self->{tmp}/encodes/".basename($item))) {
		    chomp;
		    if ($_ =~ /^\| \+ A track/) {
			$in = 1;
			undef $N;
			undef $T;
			undef $D;
			undef $U;
			$sub = 0;
		    }
		    elsif($in && $_ =~ /Track type: subtitles/) {
			$sub = 1;
		    }
		    elsif($in && $_ =~ /Track number: .*: (\d)\)$/) {
			$T = $1;
		    }
		    if (defined $in && $sub && $T) {
			$self->sys($self->{opt}->{mkvext}, 'tracks', "$self->{tmp}/encodes/".basename($item), "$T:$self->{tmp}/encodes/$T.ass");
			push @FS, "$self->{tmp}/encodes/$T.ass";
			undef $T;
			$in  = 0;
			$sub = 0;
		    }
		}
		$self->sys($self->{opt}->{mkvmerge}, '-o', "$self->{tmp}/encodes/fixed.".basename($item), '-S', "$self->{tmp}/encodes/".basename($item), @FS);
		$self->replace("$self->{tmp}/encodes/".basename($item), "$self->{tmp}/encodes/fixed.".basename($item));
	    }
	    less();

	    INFO "moving built file to final destination";
	    more();
	    $self->sys('/bin/mkdir', '-p', $self->{opt}->{outdir});
	    $self->sys('/bin/mv', "$self->{tmp}/encodes/".basename($item), "$self->{opt}->{outdir}/");
	    less();
	}
	$self->mktmp();
	less();
    }

    sub bitrate {
	my $self = shift;
	my $file = shift;
	my $size = int(((-s $file)/1024+.5)*1.1);
	my $br   = 2000;
	foreach my $line (split /\n/, $self->sys($self->{opt}->{ffmpeg}, '-i', $file)) {
	    if($line =~ /duration: (\d+):(\d+):(\d+\.\d+),/i) {
		my $duration = ($1*3600)+($2*60)+int($3+.5);
		$br = int(($size / $duration)+.5);
		DEBUG "duration [$1:$2:$3] = $duration seconds, fsize $size = ${br}k bitrate";
	    }
	    if($line =~ /duration.*bitrate: (\d+) k/i) {
		$br = $1;
		DEBUG "original bitrate $br";
	    }
	}
	return $br;
    }

    sub replace {
	my $self   = shift;
	my $dest   = shift;
	my $source = shift;
	$self->sys('/bin/rm', '-f', $dest);
	$self->sys('/bin/mv', $source, $dest);
    }

    sub is_linked {
	my $self   = shift;
	my $item   = shift;
	my $linked = 0;
	more();
	foreach my $line (split /\n/, $self->sys($self->{opt}->{mkvext}, 'chapters', $item)) {
	    if($line =~ /<ChapterSegmentUID/i) {
		$linked = 1;
	    }
	}
	if($linked) {
	    WARN "$item contains segmented chapters";
	}
	else {
	    INFO "$item does not contain segmented chapters";
	}
	less();
	return $linked;
    }

    sub setpart {
	my $self = shift;
	my $link = shift;
	my $file = shift;
	DEBUG "setting part $link => $file";
	$self->sys('/bin/ln', '-s', $file, "$self->{tmp}/parts/$link");
	return "$self->{tmp}/parts/$link";
    }

    sub uniquify_substyles {
	my $self = shift;
	my $S    = shift;
	my @styles;
	foreach my $T (@$S) {
	    DEBUG $T;
	    my $uniq = crc32($T);
	    open my $O, '>', "$T.new";
	    open my $F, '<', $T;
	    my $in = 0;
	    my $di = 0;
	    my $key;
	    while(my $line = <$F>) {
		if ($line =~ /^\[/ && $line =~ /^\[V4\+ Styles/) {
		    $in = 1;
		}
		elsif(($in||$di) && !defined $key && $line =~ /^Format:/i) {
		    my $test = "$line";
		    $test =~ s/ //g;
		    $test =~ s/^format://i;
		    my(@parts) = split /,/, $test;
		    my $c      = 0;
		    foreach my $part (@parts) {
			if ($in && $part =~ /^name$/i) {
			    $key = $c;
			}
			elsif($di && $part =~ /^style$/i) {
			    $key = $c;
			}
			$c++;
		    }
		}
		elsif($in && defined $key && $line =~ /^style:/i) {
		    $line =~ s/^style:\s+?//i;
		    my(@parts) = split /,/, $line;
		    $parts[$key] = "$parts[$key] u$uniq";
		    $line = "Style: " . join(',', @parts);
		    push @styles, $line;
		    DEBUG $line;
		}
		elsif($line =~ /^\[Events/i) {
		    $in  = 0;
		    $di  = 1;
		    $key = undef;
		}
		elsif($di && defined $key && $line =~ /^dialogue:/i) {
		    $line =~ s/^dialogue: //i;
		    my(@parts) = split /,/, $line;
		    $parts[$key] = "$parts[$key] u$uniq";
		    $line = "Dialogue: " . join(',', @parts);
		}
		print $O $line;
	    }
	    close $F;
	    close $O;
	    $self->sys('/bin/mv', '-f', "$T.new", $T);
	}
	return \@styles;
    }

    sub mush_substyles {
	my $self   = shift;
	my $S      = shift;
	my $styles = shift;
	foreach my $T (@$S) {
	    open my $F, '<', $T;
	    my @lines = <$F>;
	    close $F;
	    open my $F, '>', $T;
	    my $in = 0;
	    foreach my $line (@lines) {
		if ($line =~ /^\[/ && $line =~ /^\[V4\+ Styles/) {
		    $in = 1;
		    print $F $line;
		}
		elsif($in && $line =~ /^format:/i) {
		    print $F $line;
		    do {
			print $F $_;
		    } for @$styles;
		}
		elsif($in && $line =~ /^style:/i) {
		    #do nothing
		}
		elsif($in && $line =~ /^\[/) {
		    $in = 0;
		    print $F $line;
		}
		elsif(defined $self->{opt}->{playresx} && $line =~ /^PlayResX:/) {
		    print $F "PlayResX: $self->{opt}->{playresx}\n";
		}
		elsif(defined $self->{opt}->{playresy} && $line =~ /^PlayResY:/) {
		    print $F "PlayResY: $self->{opt}->{playresy}\n";
		}
		else {
		    print $F $line;
		}
	    }
	    close $F;
	}
    }

    sub mkvinfo {
	my $self = shift;
	my $file = shift;
	my $id  = '';
	my $dur = '';
	for(split /\n/, $self->sys($self->{opt}->{mkvinfo}, $file)) {
	    chomp;
	    if ($_ =~ /Segment UID:/) {
		$_ =~ /Segment UID:([a-zA-Z0-9\s]+)$/;
		$id = $1;
		$id =~ s/^\s+//g;
		$id =~ s/\s+$//g;
	    }
	    elsif($_ =~ /^\| \+ Duration:/) {
		$_   =~ /^\| \+ Duration:.*\((.*)\)/;
		$dur = $1;
	    }
	}
	return ($id, $dur);
    }

    sub add_duration_to_timecode {
	my $self = shift;
	my $time = shift;
	my $dur  = shift;
	my ($th, $tm, $ts) = split /:/,  $time;
	my ($dh, $dm, $ds) = split /:/,  $dur;
	$ts += ($ds + 0.000000001);
	if($ts >= 60.000000000) {
	    $ts = $ts - 60.000000000;
	    $dm++;
	}
	$tm += $dm;
	if($tm >= 60) {
	    $tm = $tm - 60;
	    $dh++;
	}
	$th += $dh;
	return sprintf("%02d:%02d:%02.9f",$th,$tm,$ts);
    }

    sub sys {
	my $self = shift;
	my $app  = shift;
	my ($pid, $in, $out, $err, $sel, $buf);
	$err = gensym();
	more();
	TRACE "sys > $app @_";
	$pid = open3($in, $out, $err, $app, @_) or LOGDIE "failed to open $app: @_";
	$sel = new IO::Select;
	$sel->add($out,$err);
	SYSLOOP: while(my @ready = $sel->can_read) {
	    foreach my $fh (@ready) {
		my $line = <$fh>;
		if(not defined $line) {
		    $sel->remove($fh);
		    next;
		}
		if($fh == $out) {
		    TRACE "sys < $line";
		    $buf .= $line;
		}
		elsif($fh == $err) {
		    TRACE "sys !! $line";
		    $buf .= $line;
		}
		else {
		    ERROR "Shouldn't be here\n";
		    return undef;
		}
	    }
	}
	waitpid($pid, 0);
	less();
	return $buf;
    }

    sub more {
	Log::Log4perl::NDC->push("  ");
    }

    sub less {
	Log::Log4perl::NDC->pop();
    }

}