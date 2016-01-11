# How to improve Koha speed?

These tests have been generated on a master branch (01d48a78258f98f693a8f48d7294a6ae880d5fc4)

And patches from [bug 15294](http://bugs.koha-community.org/bugzilla3/show_bug.cgi?id=15294) and [bug 15295](http://bugs.koha-community.org/bugzilla3/show_bug.cgi?id=15295) to have the Koha::Librar[y|ies] modules.

Using the [ansible branch](https://github.com/digibib/kohadevbox/tree/ansible) of kohadevbox, I have added the following to the file `/etc/koha/sites/kohadev/plack.psgi` to profile the script executed:

    enable 'Debug',  panels => [
            qw(Environment Response Timer Memory),
            [ 'Profiler::NYTProf', exclude => [qw(.*\.css .*\.png .*\.ico .*\.js .*\.gif .*404\.pl .*nytprof/nytprofhtml.*)] ],
    ];

Then

    sudo koha-plack --enable kohadev
    sudo koha-plack --start kohadev
    sudo service apache2 restart

Important note: Between each step koha-plack has been restarted!

## Koha::Objects, an overhead?

### Fetch 3 libraries

The following script will only fetch 3 library names:

    #!/usr/bin/perl
    
    use Modern::Perl;
    use CGI;
    use Koha::Libraries;
    for my $l (qw(CPL MPL IPT )) {
        Koha::Libraries->find($l)->branchname;
    }
    
    my $query = CGI->new;
    print $query->header({
        type => 'text/html',
        status => '200 OK',
        charset => 'UTF-8',
        Pragma => 'no-cache',
    });
    print "<html><body>Hello</body></html>";

The [profile](nytprofhtml.18554/index.html) shows that the big part of the time is spent to load the DBIx::Schema.

### Fetch a lot of libraries

The following script will fetch 3x 1000 libraries

    #!/usr/bin/perl
    
    use Modern::Perl;
    use CGI;
    use Koha::Libraries;
    for ( 1 .. 1000 ) {
        for my $l (qw(CPL MPL IPT )) {
            Koha::Libraries->find($l)->branchname;
        }
    }
    
    my $query = CGI->new;
    print $query->header({
        type => 'text/html',
        status => '200 OK',
        charset => 'UTF-8',
        Pragma => 'no-cache',
    });
    print "<html><body>Hello</body></html>";


The [profile](nytprofhtml.19366/index.html) highlights that the time spent to load the schema is always the same (1.3s).

### Fetching a lot of libraries, without Koha::Objects

Is there an issue in Koha::Objects here?

The following script does the same as before, but without using the Koha::Libraries module.

    #!/usr/bin/perl
    
    use Modern::Perl;
    use CGI;
    use Koha::Database;
    for ( 1 .. 1000 ) {
        for my $l (qw(CPL MPL IPT )) {
            Koha::Database->new->schema->resultset('Branch')->find($l)->branchname;
        }
    }
    
    my $query = CGI->new;
    print $query->header({
        type => 'text/html',
        status => '200 OK',
        charset => 'UTF-8',
        Pragma => 'no-cache',
    });
    print "<html><body>Hello</body></html>";

The [profile](nytprofhtml.19609) proves that the Koha::Objects does not add a big overhead here.

At least for ->find.

But a lot of time is spent fetching the same result, is it normal?

Update: I have configured mysql to log all queries in a log file. And the queries don't seem to be cached, there are 3000 queries of this kind:

    SELECT me.branchcode, me.branchname, me.branchaddress1, me.branchaddress2, me.branchaddress3, me.branchzip, me.branchcity, me.branchstate, me.branchcountry, me.branchphone, me.branchfax, me.branchemail, me.branchreplyto, me.branchreturnpath, me.branchurl, me.issuing, me.branchip, me.branchprinter, me.branchnotes, me.opac_info
    FROM branches me
    WHERE ( me.branchcode = 'MPL' )

### What about the ->search method?

#### With Koha::Libraries (and so Koha::Objects)

The following code will search for all libraries in the database (12) 1000 times and fetch the branchname one by one.

    #!/usr/bin/perl
    
    use Modern::Perl;
    use CGI;
    use Koha::Libraries;
    for ( 1 .. 1000 ) {
        my $libraries = Koha::Libraries->search;
        while ( my $l = $libraries->next ) {
            $l->branchname;
        }
    }
    
    my $query = CGI->new;
    print $query->header({
        type => 'text/html',
        status => '200 OK',
        charset => 'UTF-8',
        Pragma => 'no-cache',
    });
    print "<html><body>Hello</body></html>";

The [profile](nytprofhtml.20066).


#### Without Koha::Objects

    #!/usr/bin/perl
    
    use Modern::Perl;
    use CGI;
    use Koha::Database;
    for ( 1 .. 1000 ) {
        my $libraries = Koha::Database->new->schema->resultset('Branch')->search;
        while ( my $l = $libraries->next ) {
            $l->branchname;
        }
    }
    
    my $query = CGI->new;
    print $query->header({
        type => 'text/html',
        status => '200 OK',
        charset => 'UTF-8',
        Pragma => 'no-cache',
    });
    print "<html><body>Hello</body></html>";

Once again the [profile](nytprofhtml.19928) shows us that Koha::Objects does not introduce a big overhead here.

### But... Is there something magic?

And what happens if I profile twice the same script without restarting plack?

You know, the real life.

Tested with one of the previous script (not sure which one...)

The profiles are completely different: [The first call](nytprofhtml.20401) has the same graph as the previous ones.

But the [second one](nytprofhtml.20401_2) is completely different and yes, here is the magic of plack! :)

### The real real life

Ok but there is no point to call 3x 1000 times the same library names, so let's try with the main page:

Here we are, the [first profile](nytprofhtml.21326_1) confirm that the first call will load the schema and not the others.

Indeed, in the [second profile](nytprofhtml.21326_2), the DBIx::Class:Schema::load_namespaces is lost somehwere when it tooks 70% of the time in the first call.

However, there is now 60% spent in get_template_and_user, then 30% fetching syspref (???).

For a total of 315 fetches:

    # spent 777ms (10.5+767) within DBIx::Class::ResultSet::single which was called 315 times, avg 2.47ms/call: # 315 times (10.5ms+767ms) by C4::Context::preference or C4::Languages::getTranslatedLanguages or C4::Languages::getlanguage or C4::NewsChannels::GetNewsToDisplay at line 910, avg 2.47ms/call

I have a patch in the pipe, see [bug 15341](http://bugs.koha-community.org/bugzilla3/show_bug.cgi?id=15341), so let's apply it.

[The profile is here](nytprofhtml.23030)
