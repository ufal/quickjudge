#!/usr/bin/perl
# QuickJudge, a tool for manual scoring of text segments
# Ondrej Bojar, obo@cuni.cz
# http://ufal.mff.cuni.cz/euromatrix/quickjudge/
#
# Given any number of files as args, 'quickjudge' produces a file for manual
# annotation. Every input file represents a "system" and the question is which
# system delivers the best results. All input files have to have the same number
# of lines!
#
# After the file has been annotated, use quickjudge --print to read and
# de-scramble the lines.
#
# $Id: quickjudge 696 2010-11-05 08:03:32Z bojar $
#

use strict; use warnings;
use IO::File;
use Getopt::Long;

my $anotdelim = "\t"; # prefix all lines with
my $refs = undef; # comma-delimited list of reference files
my $print = 0; # print statistics from an annotation
my $verbatim = 0; # when printing, print the lines verbatim, not just the tokens
		  # from the first column
my $keep_identical = 0; # annotate even blocks where all the systems produced
                        # identical outputs
my $shuffle = 0; # shuffle output blocks (systems shuffled always)
my $portion_size = 0;
	# if nonzero, the input stream is segmented to portions of this size
my $portion_overlap = 0;
	# if nonzero, portions overlap:
	#    ----  # portion size of 4
	#       ---- # overlap of 1
my $circular = 0;
	# add $portion_overlap of sentences from front to the end as well
my $portion_num_width = 2;
	# number of digits in the portion index
my $usage = 0;
my $anotfile = undef;
GetOptions(
  "anotdelim=s" => \$anotdelim,
  "anotfile=s" => \$anotfile,
  "refs=s" => \$refs,
  "print" => \$print,
  "verbatim" => \$verbatim,
  "keep-identical" => \$keep_identical,
  "help" => \$usage,
  "shuffle" => \$shuffle,
  "portion-size=i" => \$portion_size,
  "portion-overlap=i" => \$portion_overlap,
  "portion-num-width=i" => \$portion_num_width,
  "circular" => \$circular,
) or exit 1;

$anotdelim =~ s/\\t/\t/g;
$anotdelim =~ s/\\\\/\\/g;

my $fileprefix = shift;
if (!defined $fileprefix || (0 == scalar @ARGV && !$print) || $usage) {
  print STDERR "usage:
1. Prepare file for annotation:
     quickjudge fileprefix FILE1 FILE2 ...
2. Manually edit fileprefix.anot
3. Print the results:
     quickjudge fileprefix --print [--anotfile=FN]
Options:
  --print            ... read results from manual annotation
  --anotfile=FN      ... read results from a different file, not fileprefix.anot
  --verbatim         ... print annotation lines verbatim, not just tokens
  --anotdelim='\\t'   ... prefix all lines in anot. file with this
  --keep-identical   ... annotate even blocks where all systems produced
                         identical output
  --refs  ... comma-delimited list of reference lines, not judged, not shuffled
  --shuffle          ... shuffle output blocks (systems shuffled always)
                         shuffle is not compatible with --portion*
  --portion-size=N   ... split input into pieces each of up to N lines
  --portion-num-width=2  for the filename
  --portion-overlap=N .. the pieces overlap as a chain by N lines for double
                         annotation
  --circular         ... the overlapping is circular, last portion gets the
                         first --portion-overlap lines appended
";
  exit (!$usage);
}

if ($print) {
  $fileprefix =~ s/\.anot$// if ! -e $fileprefix.".anot";
  my $anotf = $fileprefix.".anot";
  $anotf = $anotfile if defined $anotfile;
  my $corespf = $fileprefix.".coresp";
  $anotf .= ".gz" if ! -e $anotf && -e $anotf.".gz";
  $anotf .= ".bz2" if ! -e $anotf && -e $anotf.".bz2";
  *ANOTF = my_open($anotf);
  $corespf .= ".gz" if ! -e $corespf && -e $corespf.".gz";
  $corespf .= ".bz2" if ! -e $corespf && -e $corespf.".bz2";
  *CORF = my_open($corespf);

  $_ = <CORF>;
  die "Malformed $corespf" if ! /Reference lines: ([0-9]+)/;
  my $reflines = $1;
  print STDERR "Will skip first $reflines lines of every block (these are refereces).\n";
  $_ = <CORF>;

  my $nr = undef;
  my $totalblocks = 0;
  my @systems = ();
  while (<CORF>) {
    if (/^$/) {
      # end of the block
      $totalblocks++;
      # first skip reflines
      map { my $s = <ANOTF>; $s; } (0..($reflines-1));
      # read the annotations
      my @annots = map {
                  my $l = <ANOTF>;
                  die "File too short: $anotf" if !defined $l;
                  chomp $l;
                  $l;
                }
                (0..$#systems);
      for (my $i=0; $i<@annots; $i++) {
        my $line = $annots[$i];
	if ($verbatim) {
	  # print annotations verbatim, don't collect tokens
          foreach my $s (@{$systems[$i]}) {
            print "$nr\t$s\t$line\n";
          }
	} else {
	  # collect tokens from the first col only
          my ($a, $b) = split /$anotdelim/, $line;
          next if !defined $a || $a eq "";
          foreach my $s (@{$systems[$i]}) {
            print "$nr\t$s\t$a\n";
          }
	}
      }
      my $skipblank = <ANOTF>;
      @systems = ();
      $nr = undef;
    } else {
      chomp;
      if (!defined $nr) {
        # this is the line number
        $nr = $_;
      } else {
        # this is a specification of the systems
        push @systems, [split /\t/, $_];
        # print "SYSTEMS: $_\n";
      }
    }
  }
  die "Too many lines in $anotf" if !eof(ANOTF);
  print STDERR "Processed $totalblocks blocks.\n";

  exit 0;
}

my @refs;
@refs = split /,/, $refs if defined $refs;

my $portion_number;
if ($portion_size) {
  $portion_number = 1;
}

my $anotf = "no-output-filename-yet";
my @anotoutputblocks = ();
my @corespoutputblocks = ();

sub portion_file_suffix {
  my $out = "";
  $out = sprintf(".%0${portion_num_width}i", $portion_number)
    if $portion_size;
  return $out;
}
sub start_output_files {
  $anotf = $fileprefix.portion_file_suffix().".anot";
  die "Output file with annotations already exists: $anotf, will not overwrite."
    if -e $anotf;
  my $corespf = $fileprefix.portion_file_suffix().".coresp";

  open ANOTF, ">$anotf" or die "Can't write $anotf";
  open CORF, ">$corespf" or die "Can't write $corespf";
  print CORF "Reference lines: ".(scalar @refs)."\n\n";
  @anotoutputblocks = ();
  @corespoutputblocks = ();
}
sub stop_output_files {
  my @printanotoutputblocks;
  my @printcorespoutputblocks;
  if ($shuffle) {
    my $indices = shuffle([0..$#anotoutputblocks]);
    foreach my $i (@$indices) {
      push @printanotoutputblocks, $anotoutputblocks[$i];
      push @printcorespoutputblocks, $corespoutputblocks[$i];
    }
  } else {
    @printanotoutputblocks = @anotoutputblocks;
    @printcorespoutputblocks = @corespoutputblocks;
  }
  print ANOTF join("", @printanotoutputblocks);
  close ANOTF;
  print CORF join("", @printcorespoutputblocks);
  close CORF;
  $portion_number++;

  print STDERR "Now edit: $anotf\n";
  print STDERR "Print annotations using: quickjudge --print $fileprefix"
    .portion_file_suffix()."\n";
}

start_output_files();

my @files = @ARGV;
unshift @files, @refs;
my @streams = map {
  my $openstr = ($_ =~ /\.gz$/ ? "zcat $_ |" : "< $_");
  IO::File->new($openstr) or die "Can't open '$openstr'"
} @files;

my $curr_portion_size = 0;
my $circular_anot = "";
my $circular_corf = "";
my $overlap_anot = "";
my $overlap_corf = "";
my $nr = 0;
while (1) {
  $nr++;
  my $eofs = 0;
  my @block = ();
  my $all_same = 1;
  my $systems = undef;
  my $references = "";
  for(my $i=0; $i<=$#streams; $i++) {
    my $line;
    if (eof($streams[$i])) {
      die "File $files[$i] too short!";
    } else {
      $line = readline($streams[$i]);
      chomp $line;
      $line =~ s/\s+/ /g;
      $line =~ s/^\s+|\s+$//g;
    }
    $line .= "\n";

    if ($i < scalar @refs) {
      $references .= $refs[$i].$anotdelim.$line;
    } else {
      # this is a regular system to randomize
      push @{$systems->{$line}}, $files[$i];
        # which systems produced this output
    }
    $eofs++ if eof($streams[$i]);
  }
  if ($keep_identical || 1 < scalar keys %$systems
      || 1 == scalar @ARGV) {
    # print the block if all blocks wanted or if there's more than one unique
    my $emitanot = "";
    my $emitcorf = "";
    $emitanot .= $references;
    $emitcorf .= $nr."\n";
    my $randoutputs = shuffle([keys %$systems]);
    foreach my $output (@$randoutputs) {
      $emitanot .= $anotdelim.$output;
      $emitcorf .= join("\t", @{$systems->{$output}})."\n";
    }
    $emitanot .= "\n";
    $emitcorf .= "\n";
    push @anotoutputblocks, $emitanot;  # output for ANOTF
    push @corespoutputblocks, $emitcorf; # output for CORF
    # also store the bits if we need for portions
    if ($portion_size) {
      if ($circular && $nr <= $portion_overlap) {
        # remember the beginning of the file, to reuse at the end
        $circular_anot .= $emitanot;
        $circular_corf .= $emitcorf;
      }
      if ($portion_overlap
         && ($portion_size-$curr_portion_size <= $portion_overlap)) {
        # remember the tail of this portion
        $overlap_anot .= $emitanot;
        $overlap_corf .= $emitcorf;
      }

      $curr_portion_size++;
      if ($curr_portion_size >= $portion_size) {
        # just finished a portion
	stop_output_files();
	start_output_files();
	if ($portion_overlap) {
	  print ANOTF $overlap_anot;
	  print CORF $overlap_corf;
	  $overlap_anot = "";
	  $overlap_corf = "";
	}
	$curr_portion_size = $portion_overlap;
      }
    }
  }
  last if $eofs == scalar @streams;
}

if ($circular) {
  print ANOTF $circular_anot;
  print CORF $circular_corf;
}
stop_output_files();

sub shuffle {
  my $lines = shift;
  my @lines = @$lines;
  my @out = ();
  while (@lines) {
    my $rnd = int(rand($#lines+1));
    push @out, $lines[$rnd];
    splice(@lines, $rnd, 1);
  }
  return [@out];
}


sub my_open {
  my $f = shift;
  if ($f eq "-") {
    # binmode(STDIN, ":utf8");
    return *STDIN;
  }

  die "Not found: $f" if ! -e $f;

  my $opn;
  my $hdl;
  my $ft = `file '$f'`;
  # file might not recognize some files!
  if ($f =~ /\.gz$/ || $ft =~ /gzip compressed data/) {
    $opn = "zcat '$f' |";
  } elsif ($f =~ /\.bz2$/ || $ft =~ /bzip2 compressed data/) {
    $opn = "bzcat '$f' |";
  } else {
    $opn = "$f";
  }
  open $hdl, $opn or die "Can't open '$opn': $!";
  # binmode $hdl, ":utf8";
  return $hdl;
}
