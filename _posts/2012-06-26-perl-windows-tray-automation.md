---
layout: post
title:  "Perl Windows - Grab all the system tray icons & get their co-ordinates"
date:   2012-06-26 17:34:00
categories: Perl Windows Automation
---

While automating applications on Windows, every now & then I run in to a situation where I have to find a tray icon & do some mouse operations on it. And for the most part, I overlooked it as it was just a small part of my automation needs. Well, it was time to scratch that small itch as it was bothering me. So with the help of [Sinan on StackOverflow][sinan-so], I ended up writing this script.

This should work on both 32 & 64-bit platforms:

{% highlight perl %}
use strict;
use warnings;

use Data::Dumper;
use Win32::API;
use Win32::OLE qw(in);
use Win32::OLE::Variant;
use Win32::GuiTest qw(
    AllocateVirtualBuffer
    FreeVirtualBuffer
    ReadFromVirtualBuffer
    FindWindowLike
    SendMessage
);

# Used for WMI
use constant wbemFlagReturnImmediately => 0x10;
use constant wbemFlagForwardOnly       => 0x20;

# SendMessage commands
use constant TB_BUTTONCOUNT   => 0x0418;
use constant TB_GETBUTTONTEXT => 0x041B;
use constant TB_GETBUTTONINFO => 0x0441;
use constant TB_GETITEMRECT   => 0x041D;
use constant TB_GETBUTTON     => 0x0417;

sub get_windows_os_details {
    my ($self) = @_;
    my $ret;

    my $objWMIService =
      Win32::OLE->GetObject("winmgmts:\\\\localhost\\root\\CIMV2")
      or die "WMI connection failed.\n";
    my $colItems =
      $objWMIService->ExecQuery("SELECT * FROM Win32_OperatingSystem",
                               "WQL",
                               wbemFlagReturnImmediately | wbemFlagForwardOnly);

    my $objItem;
    foreach $objItem (in $colItems) {
        $ret->{'osname'} = $objItem->{Caption};
    }

    $colItems =
      $objWMIService->ExecQuery("SELECT * FROM Win32_Processor",
                               "WQL",
                               wbemFlagReturnImmediately | wbemFlagForwardOnly);

    foreach $objItem (in $colItems) {
        $ret->{'osbit'} = $objItem->{AddressWidth};
    }

    return $ret;
}

sub get_process_list {
    my $ret;
    my $class = "winmgmts:{impersonationLevel=impersonate}\\\\.\\Root\\cimv2";
    my $wmi = Win32::OLE->GetObject($class) || die;
    my $plist = $wmi->InstancesOf( "Win32_Process" );
    foreach my $proc (in($plist)) {
        $ret->{$proc->{'ProcessID'}} = $proc->{'Name'};
    }
    return $ret;
}

sub get_tray_handle {
    my ($tray_hwnd) = FindWindowLike(undef, undef, 'TrayNotifyWnd');
    my ($toolbar_hwnd) = FindWindowLike($tray_hwnd, undef, 'ToolbarWindow');
    return $toolbar_hwnd;
}

sub get_tray_icon_count {
    my $hWnd = get_tray_handle(); 
    return SendMessage($hWnd, TB_BUTTONCOUNT, 0, 0);
}


my $os = get_windows_os_details();

#http://msdn.microsoft.com/en-us/library/windows/desktop/bb760476(v=vs.85).aspx
my $tb_button;
if ($os->{'osbit'} == 64) {
    $tb_button->{'pack'} = 'i i C C A6 L L';
    $tb_button->{'size'} = 0x20; # 32 bytes
} else {
    $tb_button->{'pack'} = 'i i C C A2 L L';
    $tb_button->{'size'} = 0x1C; # 28 bytes
}


# Get tray handle 
my $tray_hwnd = get_tray_handle();
my $icon_count = get_tray_icon_count();

# All the processes & their PIDs
my $proc_list = get_process_list();

# Allocate virtual buffer for TBBUTTON Structure (depending on 32-bit on 64-bit) in
# tray process
my $buffer = AllocateVirtualBuffer($tray_hwnd, $tb_button->{'size'});
my $icons;

for (my $i=0; $i<=$icon_count; $i++) {
    # Get the button structure represented by index by using TB_GETBUTTON
    # message & then copy it to the local process space.
    my $status = SendMessage($tray_hwnd, TB_GETBUTTON, $i, $buffer->{ptr});
    my $result = ReadFromVirtualBuffer($buffer, $tb_button->{'size'});
    my ($iBitmap, $idCommand, $fsState, $fsStyle, $bReserved, $dwData, $iString) = unpack $tb_button->{'pack'}, $result;
    
    # Find the owner handle for the button
    my $read_process = Win32::API->new('kernel32', 'ReadProcessMemory', 'NNPIN','I');
    my $extra_data = pack('L2', 0, 0);
    $read_process->Call($buffer->{process}, $dwData, $extra_data, (length $extra_data), 0);
    my ($owner_hwnd, $id) = unpack('L2', $extra_data);
    
    # Find the PID of the owner process
    my $window_thread_proc_id = Win32::API->new('user32', "GetWindowThreadProcessId", 'LP', 'N');
    my $lpdwPID = pack 'L', 0;
    my $pid = $window_thread_proc_id->Call($owner_hwnd, $lpdwPID);
    my $dwPID = unpack 'L', $lpdwPID;
    
    # Find the icon coordinates
    my $rect = pack ('IIII', 0, 0 , 0, 0);
    SendMessage($tray_hwnd, TB_GETITEMRECT, $i, $buffer->{ptr});
    my $map_window_points = Win32::API->new('user32', 'MapWindowPoints', 'NIPI', 'I');
    my $ret = $map_window_points->Call($tray_hwnd, 0, $rect, 2);
    my ($left, $top, $right, $bottom) = unpack('IIII', $rect);
    
    # Fill our icons hash list
    if (defined $proc_list->{$dwPID}) {
        $icons->{$proc_list->{$dwPID}}->{'hwnd'} = $owner_hwnd;
        $icons->{$proc_list->{$dwPID}}->{'pid'} = $dwPID;
        $icons->{$proc_list->{$dwPID}}->{'x'} = $left;
        $icons->{$proc_list->{$dwPID}}->{'y'} = $top;
    }
}

# We don't need the virtual buffer any more
FreeVirtualBuffer($buffer);

print Dumper($icons);

{% endhighlight %}

[sinan-so]: http://stackoverflow.com/questions/11199287/using-perl-how-can-obtain-information-about-the-icons-in-windows-taskbar-notif
