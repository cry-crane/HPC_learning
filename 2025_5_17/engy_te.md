路径：`/share/org/PARATERA/MCC20258768/zrh/cesm1.2.1/cesm.model/models/atm/cam/src/advection/slt/engy_te.F90`  

使用MPI来优化。主要的并行优化思路是将经度(nlon)维度上的计算任务分配给不同的MPI进程。  
原代码：  
```F90

subroutine engy_te(cwava   ,w       ,t       ,u      ,v        , &
                   phis    ,pdel    ,engy    , nlon  )

!----------------------------------------------------------------------- 
! 
! Purpose: 
! Calculate contribution of current latitude to total energy
! 
! Method: 
! 
! Author: J. Olson
! 
!-----------------------------------------------------------------------
!
! $Id$
! $Author$
!
!-----------------------------------------------------------------------
!
  use shr_kind_mod, only: r8 => shr_kind_r8
  use pmgrid,       only: plev, plon
  use physconst, only: cpair

  implicit none
!
!------------------------------Arguments--------------------------------
!
  integer , intent(in)  :: nlon                 ! longitude dimension  
  real(r8), intent(in)  :: cwava                ! normalization factor    l/(g*plon)
  real(r8), intent(in)  :: w                    ! gaussian weight this latitude
  real(r8), intent(in)  :: t   (plon,plev)      ! temperature
  real(r8), intent(in)  :: u   (plon,plev)      ! u-component
  real(r8), intent(in)  :: v   (plon,plev)      ! v-component
  real(r8), intent(in)  :: phis(plon)           ! Geopotential
  real(r8), intent(in)  :: pdel(plon,plev)      ! pressure diff between interfaces
  real(r8), intent(out) :: engy                 ! accumulator
!
!---------------------------Local variables-----------------------------
!
  integer i,k               ! longitude, level indices
  real(r8) const            ! temporary constant
!
!-----------------------------------------------------------------------
!
! Integration factor (the 0.5 factor arises because gaussian weights sum to 2)
!
  const = cwava*w*0.5_r8
  engy = 0._r8
!
  do k=1,plev
     do i=1,nlon
        engy = engy + ( cpair*t(i,k)                          + &
                        0.5_r8*( u(i,k)*u(i,k) + v(i,k)*v(i,k) ) + &
                        phis(i) )*pdel(i,k)
     end do
  end do

  engy = engy*const

  return
end subroutine engy_te

```

修改后的代码：  
```f90
subroutine engy_te_mpi(cwava, w, t, u, v, phis, pdel, engy, nlon)

!-----------------------------------------------------------------------
!
! Purpose:
! Calculate contribution of current latitude to total energy using MPI
!
! Method:
! Distribute the longitude dimension across MPI processes.
!
! Author: J. Olson (modified for MPI)
!
!-----------------------------------------------------------------------
!
! $Id$
! $Author$
!
!-----------------------------------------------------------------------
!
  use shr_kind_mod, only: r8 => shr_kind_r8
  use pmgrid,        only: plev, plon
  use physconst,     only: cpair
  use mpi

  implicit none
!
!------------------------------Arguments--------------------------------
!
  integer , intent(in)  :: nlon                      ! longitude dimension
  real(r8), intent(in)  :: cwava                     ! normalization factor   l/(g*plon)
  real(r8), intent(in)  :: w                         ! gaussian weight this latitude
  real(r8), intent(in)  :: t   (:,:)                 ! temperature (plon,plev)
  real(r8), intent(in)  :: u   (:,:)                 ! u-component (plon,plev)
  real(r8), intent(in)  :: v   (:,:)                 ! v-component (plon,plev)
  real(r8), intent(in)  :: phis(:)                   ! Geopotential (plon)
  real(r8), intent(in)  :: pdel(:,:)                 ! pressure diff between interfaces (plon,plev)
  real(r8), intent(out) :: engy                      ! accumulator
!
!---------------------------Local variables-----------------------------
!
  integer :: i, k                            ! longitude, level indices
  real(r8) :: const                         ! temporary constant
  integer :: rank, size, i_start, i_end, local_nlon
  real(r8) :: local_engy
  integer :: ierr
!
!-----------------------------------------------------------------------
!
! Initialize MPI
!
  call MPI_Init(ierr)
  call MPI_Comm_rank(MPI_COMM_WORLD, rank, ierr)
  call MPI_Comm_size(MPI_COMM_WORLD, size, ierr)

!
! Calculate the portion of the longitude dimension for this process
!
  local_nlon = nlon / size
  i_start = rank * local_nlon + 1
  i_end = (rank + 1) * local_nlon
  if (rank == size - 1) i_end = nlon  ! Handle the remaining elements

!
! Integration factor (the 0.5 factor arises because gaussian weights sum to 2)
!
  const = cwava * w * 0.5_r8
  local_engy = 0._r8

!
! Perform the calculation for the local portion of the longitude dimension
!
  do k = 1, plev
    do i = i_start, i_end
      local_engy = local_engy + (cpair * t(i, k) + &
                                 0.5_r8 * (u(i, k) * u(i, k) + v(i, k) * v(i, k)) + &
                                 phis(i)) * pdel(i, k)
    end do
  end do

  local_engy = local_engy * const

!
! Reduce the local energies from all processes to the root process (rank 0)
!
  call MPI_Reduce(local_engy, engy, 1, MPI_REAL8, MPI_SUM, 0, MPI_COMM_WORLD, ierr)

!
! Finalize MPI
!
  call MPI_Finalize(ierr)

  return
end subroutine engy_te_mpi

```

运行结果757.964秒，有一秒的优化，更像是波动
