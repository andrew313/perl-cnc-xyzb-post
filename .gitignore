# Add any directories, files, or patterns you don't want to be tracked by version control
use experimental qw(smartmatch);    #use in the perl 5.26
use strict;
use warnings FATAL => 'all';
use Scalar::Util qw(looks_like_number);
use Switch;

#use feature "switch";
use POSIX;
use constant PI => ( 4 * atan2( 1, 1 ) );
use constant STEP => 5;
my $local_datestring = localtime();
my $outfile          = "001_XYZIJK.nc";

#输出文件句柄
open( HANDOUT, ">$outfile" ) || die "Can't open newfile: $!\n";

#以追加方式打开输出文件句柄
open( FILE, "<", "001_XYZIJK.aptsource" ) || die "Can't Read the file: $!\n";
##
#001_XYZIJK.aptsource###keywords.aptsource
#打开输入文件句柄
print HANDOUT "%O1000\n";
print HANDOUT "(POST BY ANDREW YANG USE PERLDEMO PROGRAM  )\n";
print HANDOUT "(处理时间日期为：$local_datestring)\n";
print HANDOUT "N00 G49 G54 G80 G40 G90 G23 G94 G17 G98\n";

my $rotation_angle = 0;    #B轴转角变量，Unit is degree
my $pcos;                  #计算转换矩阵过度变量
my $psin;                  #计算转换矩阵过度变量
my $multi_line = NULL;
my @do;
my $result;
my $B_Rotate;
my $line_number = STEP;

while (<FILE>) {
    my $line = $_;
    $result = $line;

    #首先匹配行尾，如果行尾为$号，则保存下本行，继续读取下一行，合并后一同处理。
    #CYCLE/DRILL,   10.000000,MMPM, 1000.000000,RAPTO,    1.000000,RTRCTO,  $
    #100.000000
    #匹配APT的连接符$，并将2行合并。行末匹配方式：/?$/,其中？为待匹配字符。
    #对特殊字符的匹配需要转义，如匹配$,需要写成\$,匹配^,需要写成\^
    if (m/\$\n$/) {
        s/\$\n$//;    #将本行的$去除。
        $multi_line = $_;

        #		$result=$multi_line;
    }
    else {
        if ( $multi_line ne NULL ) {
            $result     = $multi_line . $result;
            $multi_line = NULL;
        }

        #替换的格式说明。s/<pattern>/<replacement>/
        #对APT注释的处理；
        #刀具号的注释处理；TPRINT/T1 Face Mill D 32
        switch ($result) {
            case (m/^TPRINT/) {
                s/TPRINT\//(/;
                s/\s+$/)\n/;
                print HANDOUT $_;
            }    #快速移动处理
            case (m/^RAPID/) {
                s/RAPID/G0/;
                print HANDOUT $_;
            }    #刀具号的处理；LOADTL/1,ADJUST,1,SPINDL,   70.000000,MILL
            case (m/^LOADTL/) {
                @do = split( /\/|,|\s/, $_, 7 );
                $result = 'T' . $do[1] . "M06" . "\n";
                print HANDOUT $result;
            }

            #操作号的处理；OP_NAME/Facing.a
            case (m/^OP_NAME/) {
                s/OP_NAME\//(machining:/;
                s/\s+$/)\n/;
                print HANDOUT $_;
            }

            #进给速度的处理；FEDRAT/  300.0000,MMPM
            case (m/^FEDRAT/) {
                @do = split( /,|\s+/, $_, 3 );
                $do[1] = sprintf( "%d", $do[1] );
                $result = 'F' . $do[1] . "G1" . "\n";
                print HANDOUT $result;
            }

            #主轴的处理；SPINDL/   70.0000,RPM,CLW
            case (m/^SPINDL\//) {
                @do = split( /,|\s+/, $_, 4 );
                $do[1] = sprintf( "%d", $do[1] );
                $result = 'S' . $do[1] . "M03" . "\n";
                print HANDOUT $result;
            }

            #对GOTO的处理；
            case (m/^GOTO/) {
                @do = split( /,|\/\s/, $_, 7 );

                #使用","或”/“，或“空格”分割字符串，7个分别存入do[0...6]
                $B_Rotate       = -&Solving_B_angle;
                $rotation_angle = $B_Rotate;

           #使用负是错误的，因为求转角求反了。????到底取正还是负，还没有彻底理解
                $pcos = cos( &angle2arc($rotation_angle) );    #only test
                $psin = sin( &angle2arc($rotation_angle) );    #only test
                &trans_coordinate(@do);
                @do     = &round_num03(@do);
                $result = 'N'
                  . $line_number . ' X '
                  . $do[1] . ' Y '
                  . $do[2] . ' Z '
                  . $do[3] . ' B '
                  . $B_Rotate . "\n";
                $line_number += STEP;
                print HANDOUT $result;
            }

#一般用法： @somearray = split(/:+/, $string ); #括号可以不要。  若不指定$string, 则对默认变量$_操作， 两斜线间为分割符，可以用正则表达式，强悍异常。
#加工循环的处理；CYCLE/DRILL,->G81
#CYCLE/DRILL,   10.000000,MMPM, 1000.000000,RAPTO,    1.000000,RTRCTO,  $
#100.000000		两行，处理难度算法加大
# G81 X Y Z R F K
            case (m/^CYCLE\/DRILL/) {
                my @temp;
                @temp = split( /,/, $result, 9 );    #不要空格分割|\s+
                @temp = &round_num03(@temp);
                $temp[3] = sprintf( "%d", $temp[3] );
                $result =
                    'G81' . 'Z'
                  . $temp[1] . "R"
                  . $temp[5] . 'F'
                  . $temp[3] . 'K'
                  . $temp[7] . "\n";
                print HANDOUT $result;
            }

            #加工循环结束，CYCLE/OFF->
            case (m/^CYCLE\/OFF/) {
                s/CYCLE\/OFF/G80/;
                print HANDOUT $_;
            }

            #else {}
        }
        $result = NULL;
    }
}
close(FILE);
close(HANDOUT);
print "Program run OK!";

sub angle2arc {
    my $arc = PI * $rotation_angle / 180;    #Unit is arc
    return $arc;
}

sub trans_coordinate {
    my $x  = $do[1];
    my $y  = $do[2];
    my $z  = $do[3];
    my $x1 = $x * $pcos + 0 + $z * $psin;
    my $y1 = 0 + $y + 0;
    my $z1 = -$x * $psin + 0 + $z * $pcos;
    $do[1] = $x1;
    $do[2] = $y1;
    $do[3] = $z1;
}

sub round_num03 {
    my $element;
    foreach $element (@_) {

        #		if ( $element ne 'GOTO  ' && $element ne 'FROM  ' )
        if ( looks_like_number($element) )

          #如果看起来像数字则按3位截断
        {
            $element = sprintf( "%.3f", $element );
        }

        #如是数字,则按3位小数格式化
    }

    return @_;
}

sub Solving_B_angle {
    my $tmp;
    my $B_angle;
    my $ab_dot_product;
    my $i = 1;
    my $j = 1;
    my $k = 1;
    $i = $do[4];
    $j = $do[5];
    $k = $do[6];

    $tmp            = $i**2 + $k**2;
    $ab_dot_product = $k / sqrt( $i**2 + $k**2 );

    if ( $i == 0.0 ) {
        $B_angle = 180 * acos($ab_dot_product) / PI;
    }
    else {
        $B_angle = ( $i / abs($i) ) * 180 * acos($ab_dot_product) / PI;
    }

#($i/abs($i))是确定角度的正负值。i=0时无法处理
#强制按照数字来比较,当i小于0时角度在2，4象限，角度值取负，其实只需要在上式乘i的符号即可。使用if的方法比较笨。
#	if ( ( $i + 0 ) < ( 0 + 0 ) )
#	{
#		$B_angle = -180.0 * acos($ab_dot_product) / PI;
#	}
    $B_angle = sprintf( "%0.3f", $B_angle );
    return $B_angle;
}